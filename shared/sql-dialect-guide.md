# SQL Dialect Guide

Platform-specific DDL syntax reference for skills that generate SQL. All generated DDL must use the syntax appropriate for the target platform. The default platform is **BigQuery** unless the user specifies otherwise.

---

## Platform Overview

| Feature | BigQuery | Snowflake | Amazon Redshift | PostgreSQL |
|---|---|---|---|---|
| Auto-increment PK | `INT64` + application-generated or `GENERATE_UUID()` | `AUTOINCREMENT` or `IDENTITY` | `IDENTITY(1,1)` | `SERIAL` or `GENERATED ALWAYS AS IDENTITY` |
| String type | `STRING` | `VARCHAR(n)` or `TEXT` | `VARCHAR(n)` | `VARCHAR(n)` or `TEXT` |
| Integer type | `INT64` | `NUMBER(38,0)` or `INT` | `INTEGER` or `BIGINT` | `INTEGER` or `BIGINT` |
| Decimal type | `NUMERIC(p,s)` | `NUMBER(p,s)` | `DECIMAL(p,s)` | `NUMERIC(p,s)` |
| Timestamp type | `TIMESTAMP` | `TIMESTAMP_NTZ` | `TIMESTAMP` | `TIMESTAMP` |
| Boolean type | `BOOL` | `BOOLEAN` | `BOOLEAN` | `BOOLEAN` |
| Hash function (MD5) | `TO_HEX(MD5(col))` | `MD5(col)` | `MD5(col)` | `MD5(col)` |
| Current timestamp | `CURRENT_TIMESTAMP()` | `CURRENT_TIMESTAMP()` | `GETDATE()` | `NOW()` |
| String concatenation | `CONCAT(a, b)` or `a \|\| b` | `a \|\| b` | `a \|\| b` | `a \|\| b` |

---

## Auto-Increment / Surrogate Key Generation

### BigQuery
BigQuery does not natively enforce primary keys or auto-increment. Surrogate keys are generated in the ETL layer or using `GENERATE_UUID()`:
```sql
-- INT surrogate key: generated externally (e.g. dbt_utils.generate_surrogate_key())
-- UUID alternative:
SELECT GENERATE_UUID() AS customer_sk
```

### Snowflake
```sql
CREATE TABLE dim_customer (
  customer_sk   NUMBER(38,0) NOT NULL AUTOINCREMENT PRIMARY KEY,
  customer_bk   VARCHAR(100) NOT NULL,
  ...
);
```

### Amazon Redshift
```sql
CREATE TABLE dim_customer (
  customer_sk   BIGINT IDENTITY(1,1) NOT NULL,
  customer_bk   VARCHAR(100) NOT NULL,
  ...
);
```

### PostgreSQL
```sql
CREATE TABLE dim_customer (
  customer_sk   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_bk   VARCHAR(100) NOT NULL,
  ...
);
```

---

## Primary and Foreign Key Constraints

### BigQuery
BigQuery supports `PRIMARY KEY` and `FOREIGN KEY` as **non-enforced** constraints (for documentation only):
```sql
CREATE TABLE fct_orders_placed (
  order_sk      INT64 NOT NULL,
  customer_sk   INT64 NOT NULL,
  ...
  PRIMARY KEY (order_sk) NOT ENFORCED,
  FOREIGN KEY (customer_sk) REFERENCES dim_customer(customer_sk) NOT ENFORCED
);
```

### Snowflake
```sql
CREATE TABLE fct_orders_placed (
  order_sk      NUMBER(38,0) NOT NULL PRIMARY KEY,
  customer_sk   NUMBER(38,0) NOT NULL,
  ...
  FOREIGN KEY (customer_sk) REFERENCES dim_customer(customer_sk)
);
```

### Amazon Redshift
```sql
-- Redshift defines PKs/FKs but does not enforce them (informational only):
CREATE TABLE fct_orders_placed (
  order_sk      BIGINT NOT NULL,
  customer_sk   BIGINT NOT NULL,
  ...
  PRIMARY KEY (order_sk),
  FOREIGN KEY (customer_sk) REFERENCES dim_customer(customer_sk)
);
```

### PostgreSQL
```sql
CREATE TABLE fct_orders_placed (
  order_sk      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_sk   BIGINT NOT NULL REFERENCES dim_customer(customer_sk),
  ...
);
```

---

## Clustering / Sort Keys / Partition Keys

### BigQuery
```sql
-- Partitioning (date column required):
CREATE TABLE fct_orders_placed (
  ...
  order_date    DATE NOT NULL
)
PARTITION BY order_date
CLUSTER BY customer_sk, product_sk;
```

### Snowflake
```sql
CREATE TABLE fct_orders_placed (
  ...
)
CLUSTER BY (order_date, customer_sk);

-- Or alter after creation:
ALTER TABLE fct_orders_placed CLUSTER BY (order_date, customer_sk);
```

### Amazon Redshift
```sql
CREATE TABLE fct_orders_placed (
  order_sk      BIGINT NOT NULL,
  customer_sk   BIGINT NOT NULL DISTKEY,
  order_date    DATE NOT NULL,
  ...
)
SORTKEY (order_date, customer_sk)
DISTSTYLE KEY;
```

### PostgreSQL
```sql
-- PostgreSQL uses partial indexes and BRIN indexes for time-ordered data:
CREATE INDEX idx_orders_date ON fct_orders_placed USING BRIN (order_date);
CREATE INDEX idx_orders_customer ON fct_orders_placed (customer_sk);

-- Declarative table partitioning (PG 10+):
CREATE TABLE fct_orders_placed (
  ...
  order_date DATE NOT NULL
) PARTITION BY RANGE (order_date);
```

---

## Hash Functions for Data Vault Keys

All platforms: hash must be applied to `UPPER(TRIM(business_key))`. For NULLs, replace with sentinel `'^^'`. For composite keys, concatenate with separator `'~'`.

### BigQuery
```sql
-- Single business key:
TO_HEX(MD5(UPPER(TRIM(COALESCE(customer_id, '^^'))))) AS customer_hk

-- Composite business key:
TO_HEX(MD5(CONCAT(
  UPPER(TRIM(COALESCE(order_id, '^^'))),
  '~',
  UPPER(TRIM(COALESCE(customer_id, '^^')))
))) AS lnk_order_customer_hk
```

### Snowflake
```sql
-- Single business key:
MD5(UPPER(TRIM(COALESCE(customer_id, '^^')))) AS customer_hk

-- Composite business key:
MD5(UPPER(TRIM(COALESCE(order_id, '^^'))) || '~' || UPPER(TRIM(COALESCE(customer_id, '^^')))) AS lnk_order_customer_hk
```

### Amazon Redshift
```sql
-- Single business key:
MD5(UPPER(TRIM(COALESCE(customer_id, '^^')))) AS customer_hk

-- Composite business key:
MD5(UPPER(TRIM(COALESCE(order_id, '^^'))) || '~' || UPPER(TRIM(COALESCE(customer_id, '^^')))) AS lnk_order_customer_hk
```

### PostgreSQL
```sql
-- Single business key:
MD5(UPPER(TRIM(COALESCE(customer_id, '^^')))) AS customer_hk

-- Composite business key:
MD5(UPPER(TRIM(COALESCE(order_id, '^^'))) || '~' || UPPER(TRIM(COALESCE(customer_id, '^^')))) AS lnk_order_customer_hk
```

---

## MERGE / UPSERT Syntax

### BigQuery
```sql
MERGE target_table T
USING source_table S
ON T.surrogate_key = S.surrogate_key
WHEN MATCHED AND T.hash_diff != S.hash_diff THEN
  UPDATE SET col1 = S.col1, updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (col1, col2, ...) VALUES (S.col1, S.col2, ...);
```

### Snowflake
```sql
MERGE INTO target_table T
USING source_table S
ON T.surrogate_key = S.surrogate_key
WHEN MATCHED AND T.hash_diff != S.hash_diff THEN
  UPDATE SET col1 = S.col1, updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (col1, col2, ...) VALUES (S.col1, S.col2, ...);
```

### Amazon Redshift
Redshift does not support `MERGE`. Use the DELETE + INSERT pattern:
```sql
-- Step 1: Delete rows that will be updated
DELETE FROM target_table
USING source_table S
WHERE target_table.surrogate_key = S.surrogate_key
  AND target_table.hash_diff != S.hash_diff;

-- Step 2: Insert new and updated rows
INSERT INTO target_table (col1, col2, ...)
SELECT S.col1, S.col2, ...
FROM source_table S
LEFT JOIN target_table T ON T.surrogate_key = S.surrogate_key
WHERE T.surrogate_key IS NULL;
```

### PostgreSQL
```sql
INSERT INTO target_table (col1, col2, ...)
SELECT S.col1, S.col2, ...
FROM source_table S
ON CONFLICT (surrogate_key) DO UPDATE
  SET col1 = EXCLUDED.col1,
      updated_at = NOW()
WHERE target_table.hash_diff != EXCLUDED.hash_diff;
```

---

## SCD Type 2 Date/Boolean Sentinel Values

| Platform | Expiry date for "current" row | is_current |
|---|---|---|
| BigQuery | `DATE '9999-12-31'` | `TRUE` |
| Snowflake | `'9999-12-31'::DATE` | `TRUE` |
| Redshift | `'9999-12-31'::DATE` | `TRUE` |
| PostgreSQL | `'9999-12-31'::DATE` | `TRUE` |

---

## String Functions Quick Reference

| Operation | BigQuery | Snowflake | Redshift | PostgreSQL |
|---|---|---|---|---|
| Upper case | `UPPER(col)` | `UPPER(col)` | `UPPER(col)` | `UPPER(col)` |
| Trim whitespace | `TRIM(col)` | `TRIM(col)` | `TRIM(col)` | `TRIM(col)` |
| String length | `LENGTH(col)` | `LENGTH(col)` | `LEN(col)` | `LENGTH(col)` |
| Substring | `SUBSTR(col,1,5)` | `SUBSTR(col,1,5)` | `SUBSTRING(col,1,5)` | `SUBSTRING(col,1,5)` |
| Replace NULL | `COALESCE(col,'^^')` | `COALESCE(col,'^^')` | `COALESCE(col,'^^')` | `COALESCE(col,'^^')` |
| Cast | `CAST(col AS STRING)` | `col::VARCHAR` | `col::VARCHAR` | `col::TEXT` |
