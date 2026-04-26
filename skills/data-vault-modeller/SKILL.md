---
name: data-vault-modeller
description: Converts a relational or dimensional model into Data Vault 2.0 patterns. Generates Hub, Link, and Satellite DDL with business keys, hash keys, load metadata, and record source tracking. Use when the user wants to build a Data Vault layer from an existing schema or entity descriptions.
---

# Data Vault Modeller

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md` — definitions for Hub, Link, Satellite, Hash Key, Hash Diff, PIT, Raw Vault, Business Vault
- `shared/naming-conventions.md`
- `shared/sql-dialect-guide.md`

## Step 2: Gather Inputs

Ask for any inputs not provided:

**Required:**
1. **Source schema** — DDL or list of source tables with columns
2. **Business keys per entity** — Which column(s) uniquely identify each entity in the source system? These become Hub business keys. (e.g. "Customer is identified by `customer_id`; Product by `product_sku`; Order by `order_id`")
3. **Source system name(s)** — Names of the source systems feeding this vault (used for `record_source` values, e.g. `crm`, `erp`, `pos`)
4. **SQL dialect** — BigQuery (default), Snowflake, Redshift, PostgreSQL

**Optional:**
5. **Relationships between entities** — Which entities have explicit relationships that need Link tables? (e.g. "An order belongs to a customer and contains multiple products")
6. **PIT table requirement** — Does the user need Point-In-Time tables for efficient snapshot queries?
7. **Layer being built** — Raw Vault (source-aligned, no business rules) or Business Vault (integrated, with transformations)?

Clarify: if the user says "Data Vault", confirm whether they want Raw Vault, Business Vault, or both.

## Step 3: Identify Business Keys and Entities

For each entity in the source schema, confirm:

| Entity | Source Table | Business Key Column(s) | Composite? | Notes |
|---|---|---|---|---|
| Customer | `raw_crm.customers` | `customer_id` | No | |
| Product | `raw_erp.products` | `product_sku` | No | |
| Order | `raw_pos.orders` | `order_id` | No | |
| Order Line | `raw_pos.order_lines` | `order_id` + `line_number` | Yes — composite | Hash both columns with `~` separator |

**Flag composite business keys:** When a business key is composite, the hash key is computed from the concatenation of all component values with a `~` separator.

## Step 4: Design Hub Tables

One Hub per business entity.

**Hub table structure:**
- `<entity>_hk` — hash key (PK), CHAR(32) for MD5 or CHAR(64) for SHA-256
- `load_date` — TIMESTAMP, when this record first appeared in the vault
- `record_source` — VARCHAR, which source system provided this record
- `<entity>_bk` — the business key column(s)

**Example:**
```sql
-- hub_customer
CREATE TABLE IF NOT EXISTS hub_customer (
  customer_hk     CHAR(32)     NOT NULL,   -- MD5 hash of customer_bk
  load_date       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  record_source   VARCHAR(100) NOT NULL,
  customer_bk     VARCHAR(100) NOT NULL,
  PRIMARY KEY (customer_hk)
);

-- hub_product
CREATE TABLE IF NOT EXISTS hub_product (
  product_hk      CHAR(32)     NOT NULL,
  load_date       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  record_source   VARCHAR(100) NOT NULL,
  product_bk      VARCHAR(100) NOT NULL,   -- product_sku from source
  PRIMARY KEY (product_hk)
);

-- hub_order
CREATE TABLE IF NOT EXISTS hub_order (
  order_hk        CHAR(32)     NOT NULL,
  load_date       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  record_source   VARCHAR(100) NOT NULL,
  order_bk        VARCHAR(100) NOT NULL,
  PRIMARY KEY (order_hk)
);
```

## Step 5: Design Link Tables

One Link per relationship between two or more Hubs.

**Link table structure:**
- `lnk_<e1>_<e2>_hk` — link hash key (PK), hash of all participant business keys
- `load_date` — TIMESTAMP
- `record_source` — VARCHAR
- `<entity1>_hk` — FK to hub_entity1
- `<entity2>_hk` — FK to hub_entity2

**Example:**
```sql
-- lnk_order_customer: order belongs to a customer
CREATE TABLE IF NOT EXISTS lnk_order_customer (
  lnk_order_customer_hk   CHAR(32)     NOT NULL,
  load_date               TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  record_source           VARCHAR(100) NOT NULL,
  order_hk                CHAR(32)     NOT NULL,
  customer_hk             CHAR(32)     NOT NULL,
  PRIMARY KEY (lnk_order_customer_hk),
  FOREIGN KEY (order_hk)    REFERENCES hub_order(order_hk),
  FOREIGN KEY (customer_hk) REFERENCES hub_customer(customer_hk)
);

-- lnk_order_product: order line contains a product (3-way link for order lines)
CREATE TABLE IF NOT EXISTS lnk_order_product (
  lnk_order_product_hk    CHAR(32)     NOT NULL,
  load_date               TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  record_source           VARCHAR(100) NOT NULL,
  order_hk                CHAR(32)     NOT NULL,
  product_hk              CHAR(32)     NOT NULL,
  PRIMARY KEY (lnk_order_product_hk),
  FOREIGN KEY (order_hk)   REFERENCES hub_order(order_hk),
  FOREIGN KEY (product_hk) REFERENCES hub_product(product_hk)
);
```

## Step 6: Design Satellite Tables

One or more Satellites per Hub or Link. One Satellite per source system per entity.

**Satellite structure:**
- Parent hash key (FK) — `customer_hk`, `order_hk`, etc.
- `load_date` — part of PK; when this attribute version was loaded
- `load_end_date` — NULL for current record; populated when superseded
- `record_source` — source system
- `hash_diff` — hash of all descriptive attribute values; used to detect changes
- All descriptive attribute columns

**Example:**
```sql
-- sat_customer_crm: descriptive attributes of customer from CRM
CREATE TABLE IF NOT EXISTS sat_customer_crm (
  customer_hk     CHAR(32)     NOT NULL,
  load_date       TIMESTAMP    NOT NULL,
  load_end_date   TIMESTAMP,              -- NULL = current record
  record_source   VARCHAR(100) NOT NULL,
  hash_diff       CHAR(32)     NOT NULL,  -- MD5 of all attribute values
  customer_name   VARCHAR(200),
  email           VARCHAR(200),
  region          VARCHAR(100),
  segment         VARCHAR(50),
  PRIMARY KEY (customer_hk, load_date),
  FOREIGN KEY (customer_hk) REFERENCES hub_customer(customer_hk)
);

-- sat_order_pos: attributes of order from POS system
CREATE TABLE IF NOT EXISTS sat_order_pos (
  order_hk        CHAR(32)     NOT NULL,
  load_date       TIMESTAMP    NOT NULL,
  load_end_date   TIMESTAMP,
  record_source   VARCHAR(100) NOT NULL,
  hash_diff       CHAR(32)     NOT NULL,
  order_status    VARCHAR(50),
  total_amount    NUMERIC(18,2),
  currency_code   CHAR(3),
  created_at      TIMESTAMP,
  PRIMARY KEY (order_hk, load_date),
  FOREIGN KEY (order_hk) REFERENCES hub_order(order_hk)
);
```

## Step 7: Generate Hash Key SQL

Provide the exact SQL expressions for computing hash keys, per dialect. Choose either MD5 (CHAR(32)) or SHA-256 (CHAR(64)) — state the choice explicitly and use it consistently.

**Default recommendation: MD5** (CHAR(32), faster, sufficient for most vaults; use SHA-256 if collision risk is a concern for very high cardinality business keys).

### Single Business Key (BigQuery)
```sql
TO_HEX(MD5(UPPER(TRIM(COALESCE(CAST(customer_id AS STRING), '^^'))))) AS customer_hk
```

### Composite Business Key (BigQuery)
```sql
TO_HEX(MD5(
  CONCAT(
    UPPER(TRIM(COALESCE(CAST(order_id AS STRING), '^^'))),
    '~',
    UPPER(TRIM(COALESCE(CAST(line_number AS STRING), '^^')))
  )
)) AS order_line_hk
```

### Link Hash Key (BigQuery)
```sql
-- Hash of all participant business keys concatenated
TO_HEX(MD5(
  CONCAT(
    UPPER(TRIM(COALESCE(CAST(order_id AS STRING), '^^'))),
    '~',
    UPPER(TRIM(COALESCE(CAST(customer_id AS STRING), '^^')))
  )
)) AS lnk_order_customer_hk
```

### Hash Diff (BigQuery)
```sql
-- Hash of all descriptive attribute values
TO_HEX(MD5(
  CONCAT(
    COALESCE(UPPER(TRIM(customer_name)), '^^'), '~',
    COALESCE(UPPER(TRIM(email)), '^^'), '~',
    COALESCE(UPPER(TRIM(region)), '^^'), '~',
    COALESCE(UPPER(TRIM(segment)), '^^')
  )
)) AS hash_diff
```

Provide equivalent SQL for the user's target dialect using `shared/sql-dialect-guide.md`.

## Step 8: Generate Load Patterns

### Hub Load (INSERT if not exists)
```sql
INSERT INTO hub_customer (customer_hk, load_date, record_source, customer_bk)
SELECT DISTINCT
  TO_HEX(MD5(UPPER(TRIM(COALESCE(CAST(customer_id AS STRING), '^^'))))) AS customer_hk,
  CURRENT_TIMESTAMP() AS load_date,
  'crm' AS record_source,
  CAST(customer_id AS STRING) AS customer_bk
FROM raw_crm.customers AS src
WHERE NOT EXISTS (
  SELECT 1 FROM hub_customer h WHERE h.customer_hk = TO_HEX(MD5(UPPER(TRIM(COALESCE(CAST(src.customer_id AS STRING), '^^')))))
);
```

### Satellite Load (INSERT if hash_diff changed)
```sql
INSERT INTO sat_customer_crm (customer_hk, load_date, record_source, hash_diff, customer_name, email, region, segment)
SELECT
  TO_HEX(MD5(UPPER(TRIM(COALESCE(CAST(src.customer_id AS STRING), '^^'))))) AS customer_hk,
  CURRENT_TIMESTAMP() AS load_date,
  'crm' AS record_source,
  TO_HEX(MD5(CONCAT(
    COALESCE(UPPER(TRIM(src.customer_name)), '^^'), '~',
    COALESCE(UPPER(TRIM(src.email)), '^^'), '~',
    COALESCE(UPPER(TRIM(src.region)), '^^'), '~',
    COALESCE(UPPER(TRIM(src.segment)), '^^')
  ))) AS hash_diff,
  src.customer_name,
  src.email,
  src.region,
  src.segment
FROM raw_crm.customers AS src
LEFT JOIN (
  SELECT customer_hk, hash_diff
  FROM sat_customer_crm
  WHERE load_end_date IS NULL  -- latest record per customer
) AS latest ON TO_HEX(MD5(UPPER(TRIM(COALESCE(CAST(src.customer_id AS STRING), '^^'))))) = latest.customer_hk
WHERE latest.customer_hk IS NULL           -- new customer
   OR latest.hash_diff != TO_HEX(MD5(...));  -- attributes changed
```

## Step 9: Optional — Point-In-Time (PIT) Table

If the user requires snapshot queries across multiple Satellites efficiently:

```sql
CREATE TABLE IF NOT EXISTS pit_customer (
  customer_hk         CHAR(32)  NOT NULL,
  snapshot_date       DATE      NOT NULL,
  -- For each satellite, the latest load_date on or before snapshot_date:
  sat_customer_crm_load_date   TIMESTAMP,
  sat_customer_billing_load_date TIMESTAMP,
  PRIMARY KEY (customer_hk, snapshot_date)
);
```

PIT population pattern:
```sql
INSERT INTO pit_customer
SELECT
  h.customer_hk,
  d.snapshot_date,
  MAX(CASE WHEN s1.load_date <= d.snapshot_date THEN s1.load_date END) AS sat_customer_crm_load_date,
  MAX(CASE WHEN s2.load_date <= d.snapshot_date THEN s2.load_date END) AS sat_customer_billing_load_date
FROM hub_customer h
CROSS JOIN date_spine d   -- a table of dates to snapshot
LEFT JOIN sat_customer_crm s1 ON h.customer_hk = s1.customer_hk
LEFT JOIN sat_customer_billing s2 ON h.customer_hk = s2.customer_hk
GROUP BY h.customer_hk, d.snapshot_date;
```

## Step 10: Output Summary

Produce an entity map summarising everything generated:

| Source Entity | Hub | Links | Satellites |
|---|---|---|---|
| Customer (CRM) | `hub_customer` | `lnk_order_customer` | `sat_customer_crm` |
| Product (ERP) | `hub_product` | `lnk_order_product` | `sat_product_erp` |
| Order (POS) | `hub_order` | `lnk_order_customer`, `lnk_order_product` | `sat_order_pos` |

---

## Self-Check Before Finalising

- [ ] Hash algorithm (MD5 or SHA-256) is stated explicitly and used consistently across all tables
- [ ] NULL sentinel value `'^^'` is used in all hash key expressions
- [ ] Composite business keys use `'~'` separator in the concatenation
- [ ] Hub tables have exactly: hash_key (PK), load_date, record_source, business key
- [ ] Satellite PKs are (parent_hk, load_date) — composite primary key
- [ ] `hash_diff` column is included in all Satellite tables
- [ ] Load INSERT patterns cover both "new entity" and "changed attributes" cases
- [ ] Entity map summary table is complete
- [ ] Clarification has been made: Raw Vault vs Business Vault
