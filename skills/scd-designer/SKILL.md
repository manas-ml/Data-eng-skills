---
name: scd-designer
description: Designs Slowly Changing Dimension (SCD) implementations for a given dimension entity. Produces a type recommendation, complete DDL, CDC merge/upsert SQL, dbt Snapshot YAML, and validation queries. Use when the user describes a dimension that changes over time and needs a history-tracking strategy.
---

# SCD Designer

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md` — definitions for SCD Types 1–6
- `shared/naming-conventions.md`
- `shared/sql-dialect-guide.md`

## Step 2: Gather Inputs

Ask for any inputs not provided:

**Required:**
1. **Dimension entity name** — e.g. "Customer", "Employee", "Product"
2. **Columns** — full list of columns in the current or proposed dimension table
3. **Slowly changing attributes** — which specific columns change over time and need tracking? (e.g. "address, segment change; name almost never changes")
4. **History requirement** — does the user need point-in-time historical accuracy for reporting? (e.g. "I need to know what segment a customer was in when they placed their order two years ago")
5. **Source CDC mechanism** — how does data arrive?
   - Full extract (complete table snapshot every load)
   - Delta extract (only changed rows)
   - CDC log (insert/update/delete events from database log)
6. **SQL dialect** — BigQuery (default), Snowflake, Redshift, PostgreSQL

**Optional:**
7. **Approximate change frequency** — how often do the tracked attributes change? (daily, weekly, monthly, rarely)
8. **Bi-temporal requirement** — does the user need to track both "valid time" (when the change was true in the business) and "transaction time" (when it was recorded in the system)?

## Step 3: Recommend the SCD Type

Present the decision table below and recommend the most appropriate type based on the inputs:

### SCD Type Decision Table

| If... | Recommended Type |
|---|---|
| History is irrelevant; only the current state matters | **Type 1** — Overwrite |
| Full point-in-time historical accuracy required | **Type 2** — Add row per change |
| Only "current vs previous" needed; one level of history sufficient | **Type 3** — Add previous column |
| One or more attributes change very frequently (daily or more) causing row explosion with Type 2 | **Type 4** — Mini-dimension |
| Full history required AND current value must be accessible without a subquery | **Type 6** — Hybrid (1+2+3) |

Explain the recommendation with one or two sentences referencing the user's stated requirements.

**Type 6 explanation** (include whenever Type 6 is recommended): Type 6 is called "hybrid" because it combines Type 1 (current value stamped on every row), Type 2 (new row per change with date range), and Type 3 (previous value column). The name 6 = 1 + 2 + 3.

## Step 4: Generate DDL

Generate the complete `CREATE TABLE` DDL for the chosen SCD type.

### Type 1 DDL
```sql
-- Simple dimension — no history; latest value always overwrites
CREATE TABLE IF NOT EXISTS dim_customer (
  customer_sk     INT64   NOT NULL,
  customer_bk     STRING  NOT NULL,
  customer_name   STRING,
  email           STRING,
  region          STRING,
  segment         STRING,
  _loaded_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_sk) NOT ENFORCED,
  UNIQUE KEY (customer_bk) NOT ENFORCED
);
```

### Type 2 DDL
```sql
-- Full history: one row per version of the customer
CREATE TABLE IF NOT EXISTS dim_customer_scd2 (
  customer_sk       INT64     NOT NULL,  -- unique per version; changes with each new row
  customer_bk       STRING    NOT NULL,  -- business key; stable across versions
  customer_name     STRING,
  email             STRING,
  region            STRING,              -- tracked: changes preserved
  segment           STRING,              -- tracked: changes preserved
  effective_date    DATE      NOT NULL,  -- date this version became active
  expiry_date       DATE      NOT NULL DEFAULT DATE '9999-12-31',  -- '9999-12-31' = current
  is_current        BOOL      NOT NULL DEFAULT TRUE,
  _loaded_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_sk) NOT ENFORCED
);
-- Note: customer_bk is NOT unique — multiple rows per customer (one per version)
CREATE INDEX idx_customer_bk_current ON dim_customer_scd2 (customer_bk, is_current);
```

### Type 3 DDL
```sql
-- One level of history: current + previous value columns
CREATE TABLE IF NOT EXISTS dim_customer_scd3 (
  customer_sk        INT64   NOT NULL,
  customer_bk        STRING  NOT NULL,
  customer_name      STRING,
  email              STRING,
  current_region     STRING,
  previous_region    STRING,             -- previous value of region
  region_changed_at  DATE,              -- when the region last changed
  current_segment    STRING,
  previous_segment   STRING,
  segment_changed_at DATE,
  _loaded_at         TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_sk) NOT ENFORCED,
  UNIQUE KEY (customer_bk) NOT ENFORCED
);
```

### Type 4 DDL (Mini-Dimension)
```sql
-- Main dimension: stable attributes only
CREATE TABLE IF NOT EXISTS dim_customer (
  customer_sk      INT64   NOT NULL,
  customer_bk      STRING  NOT NULL,
  customer_name    STRING,
  email            STRING,
  _loaded_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_sk) NOT ENFORCED
);

-- Mini-dimension: fast-changing attributes
CREATE TABLE IF NOT EXISTS dim_customer_profile (
  customer_profile_sk  INT64   NOT NULL,
  income_band          STRING,   -- e.g. 'LOW', 'MED', 'HIGH'
  age_band             STRING,   -- e.g. '18-25', '26-35'
  segment              STRING,
  _loaded_at           TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_profile_sk) NOT ENFORCED
);

-- Fact table references both:
-- customer_sk (main dim FK) + customer_profile_sk (mini-dim FK)
```

### Type 6 DDL
```sql
-- Hybrid: Type 2 history rows + current_<attr> stamped on all rows
CREATE TABLE IF NOT EXISTS dim_customer_scd6 (
  customer_sk        INT64     NOT NULL,
  customer_bk        STRING    NOT NULL,
  customer_name      STRING,
  email              STRING,
  region             STRING,           -- value at this version (Type 2)
  segment            STRING,           -- value at this version (Type 2)
  current_region     STRING,           -- current value stamped on ALL rows (Type 1)
  current_segment    STRING,           -- current value stamped on ALL rows (Type 1)
  previous_region    STRING,           -- previous value (Type 3)
  previous_segment   STRING,           -- previous value (Type 3)
  effective_date     DATE      NOT NULL,
  expiry_date        DATE      NOT NULL DEFAULT DATE '9999-12-31',
  is_current         BOOL      NOT NULL DEFAULT TRUE,
  _loaded_at         TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_sk) NOT ENFORCED
);
```

## Step 5: Generate MERGE / Upsert SQL

### Type 1 MERGE
```sql
MERGE INTO dim_customer AS T
USING staging_customer AS S
  ON T.customer_bk = S.customer_bk
WHEN MATCHED THEN
  UPDATE SET
    customer_name = S.customer_name,
    email         = S.email,
    region        = S.region,
    segment       = S.segment,
    _loaded_at    = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (customer_sk, customer_bk, customer_name, email, region, segment, _loaded_at)
  VALUES (S.customer_sk, S.customer_bk, S.customer_name, S.email, S.region, S.segment, CURRENT_TIMESTAMP());
```

### Type 2 MERGE
```sql
-- Step 1: Expire rows where tracked attributes have changed
MERGE INTO dim_customer_scd2 AS T
USING staging_customer AS S
  ON T.customer_bk = S.customer_bk AND T.is_current = TRUE
WHEN MATCHED AND (
  T.region != S.region OR T.segment != S.segment  -- tracked attributes changed
) THEN
  UPDATE SET
    expiry_date = CURRENT_DATE() - 1,
    is_current  = FALSE;

-- Step 2: Insert new version rows (for changed + new customers)
INSERT INTO dim_customer_scd2 (customer_sk, customer_bk, customer_name, email, region, segment, effective_date, expiry_date, is_current)
SELECT
  GENERATE_UUID() AS customer_sk,  -- new surrogate for this version
  S.customer_bk,
  S.customer_name,
  S.email,
  S.region,
  S.segment,
  CURRENT_DATE() AS effective_date,
  DATE '9999-12-31' AS expiry_date,
  TRUE AS is_current
FROM staging_customer S
LEFT JOIN dim_customer_scd2 T
  ON T.customer_bk = S.customer_bk AND T.is_current = TRUE
WHERE T.customer_bk IS NULL   -- new customers
   OR T.region != S.region    -- changed attributes
   OR T.segment != S.segment;
```

### Type 6 MERGE
After inserting the new Type 2 row (as above), update `current_region` and `current_segment` on ALL existing rows for the same `customer_bk`:
```sql
UPDATE dim_customer_scd6
SET
  current_region  = latest.region,
  current_segment = latest.segment
FROM (
  SELECT customer_bk, region, segment
  FROM dim_customer_scd6
  WHERE is_current = TRUE
) AS latest
WHERE dim_customer_scd6.customer_bk = latest.customer_bk;
```

## Step 6: Generate dbt Snapshot YAML

For Type 2 (and Type 6), provide the equivalent dbt snapshot:

```yaml
# snapshots/customer_snapshot.sql
{% snapshot customer_snapshot %}

{{
  config(
    target_schema='snapshots',
    unique_key='customer_bk',
    strategy='timestamp',          -- use 'check' if no reliable updated_at
    updated_at='updated_at',       -- source column indicating last change
    invalidate_hard_deletes=True
  )
}}

select
  customer_bk,
  customer_name,
  email,
  region,
  segment
from {{ source('<source_name>', 'customers') }}

{% endsnapshot %}
```

**Strategy guidance:**
- Use `strategy='timestamp'` only if the source has a reliable `updated_at` column that is always updated on any change to a tracked attribute
- Use `strategy='check'` with `check_cols=['region', 'segment']` if no reliable timestamp exists — dbt will hash the listed columns to detect changes
- Ask the user which applies before finalising

## Step 7: Validation Queries

Provide SQL to validate the SCD implementation after loading:

```sql
-- 1. No overlapping date ranges for the same customer_bk (Type 2)
SELECT customer_bk, COUNT(*)
FROM dim_customer_scd2
GROUP BY customer_bk, effective_date
HAVING COUNT(*) > 1;
-- Expected: 0 rows

-- 2. Exactly one is_current = TRUE per customer_bk (Type 2)
SELECT customer_bk, SUM(CASE WHEN is_current THEN 1 ELSE 0 END) AS current_count
FROM dim_customer_scd2
GROUP BY customer_bk
HAVING current_count != 1;
-- Expected: 0 rows

-- 3. No NULL effective dates
SELECT * FROM dim_customer_scd2 WHERE effective_date IS NULL;
-- Expected: 0 rows

-- 4. Expiry date is always >= effective date
SELECT * FROM dim_customer_scd2 WHERE expiry_date < effective_date;
-- Expected: 0 rows

-- 5. current_* columns match the is_current=TRUE row (Type 6)
SELECT a.customer_bk
FROM dim_customer_scd6 a
JOIN (SELECT customer_bk, region, segment FROM dim_customer_scd6 WHERE is_current = TRUE) b
  ON a.customer_bk = b.customer_bk
WHERE a.current_region != b.region OR a.current_segment != b.segment;
-- Expected: 0 rows
```

---

## Self-Check Before Finalising

- [ ] SCD type recommendation cites the user's stated requirements as justification
- [ ] Type 6 explanation is included if Type 6 is recommended
- [ ] DDL includes `effective_date`, `expiry_date`, `is_current` for all Type 2 and Type 6 designs
- [ ] MERGE SQL covers both MATCHED (change detection) and NOT MATCHED (new record insertion)
- [ ] dbt snapshot strategy (timestamp vs check) question is answered or asked
- [ ] All 5 validation queries are provided
- [ ] Dialect-specific syntax is used throughout (refer to `shared/sql-dialect-guide.md`)
