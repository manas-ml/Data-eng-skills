---
name: model-optimizer
description: Analyzes a data model and query patterns to recommend performance optimizations. Produces indexing strategies, partitioning schemes, clustering key recommendations, materialization strategies, and denormalization opportunities for a specific target platform. Use when the user provides a schema plus sample slow queries or access pattern descriptions.
---

# Model Optimizer

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md`
- `shared/sql-dialect-guide.md`

## Step 2: Gather Required Inputs

Do not proceed without all required inputs. Ask for each one that is missing.

**Required:**
1. **Schema DDL** — `CREATE TABLE` statements for all relevant tables
2. **Query patterns** — Either specific slow SQL queries or a description of access patterns (e.g. "daily report filtering by region and date range, aggregating revenue by product category")
3. **Target platform** — BigQuery, Snowflake, Amazon Redshift, or PostgreSQL

**Strongly recommended:**
4. **Approximate row counts** per table (order of magnitude is sufficient: 1K, 10K, 1M, 100M, 1B)
5. **Current performance metrics** — query runtime, bytes scanned, slot usage, or EXPLAIN output if available

**Optional:**
6. **Partition or cluster keys already in place** — to avoid redundant recommendations
7. **Materialized views or pre-aggregations already existing**

## Step 3: Analyse Query Patterns

For each query or access pattern provided, analyse:

| Query | Join Path | Filter Predicates | GROUP BY / ORDER BY | Estimated Scan |
|---|---|---|---|---|
| Query 1 | fct → dim_customer → dim_date | customer.region = 'EMEA', date between X and Y | GROUP BY product_category | Full fact scan without partition |

Identify the most expensive operations for each query:
- **Full table scans** on large fact tables without partition filter
- **High-cardinality joins** (fact to fact; or fact to un-indexed dimension)
- **Repeated aggregations** that could be pre-computed
- **Unnecessary column reads** (SELECT * on wide tables)
- **Sort-heavy operations** on unsorted tables

## Step 4: Index Recommendations

For PostgreSQL specifically (BigQuery/Snowflake/Redshift do not use traditional indexes):

For each recommended index, specify:

| Table | Column(s) | Index Type | Rationale | DDL |
|---|---|---|---|---|
| `dim_customer` | `region` | B-tree | Filter predicate on all reports | `CREATE INDEX idx_customer_region ON dim_customer (region);` |
| `fct_orders_placed` | `order_date` | BRIN | Time-ordered insert; range scans | `CREATE INDEX idx_orders_date ON fct_orders_placed USING BRIN (order_date);` |

**Index types guide:**
- **B-tree** — default; equality and range queries on high-cardinality columns
- **Bitmap** — low-cardinality columns with few distinct values (gender, status); best for OLAP read workloads
- **BRIN** — time-ordered or physically sequential columns; very small, fast for large tables
- **Hash** — equality-only lookups; no range support
- **Covering index** — includes non-key columns to satisfy a query entirely from the index (avoids heap fetch)

For BigQuery / Snowflake / Redshift: skip traditional indexes and move to Steps 5 and 6.

## Step 5: Partitioning Strategy

Recommend a partition strategy for each large fact table (>1M rows):

| Table | Recommended Partition Column | Granularity | Estimated Pruning | DDL |
|---|---|---|---|---|
| `fct_orders_placed` | `order_date` | Daily | ~97% for daily reports (only 1 of 365 partitions scanned) | See platform DDL below |

**Guidance:**
- Always partition by a date/timestamp column if one exists — it gives the highest pruning benefit for time-bounded reports
- Use daily partitions for tables with <5 years of data; monthly for longer time ranges
- Never partition on a column used only in JOINs (not filters)
- For BigQuery: `PARTITION BY DATE(order_date)` (ingestion-time or column-based)
- For Snowflake: Virtual warehouses handle partitioning automatically via micro-partitions; explicit `CLUSTER BY` is a better lever
- For Redshift: Use `DISTKEY` for the join column and `SORTKEY` for the filter column

Provide the platform-specific DDL or ALTER statement.

## Step 6: Clustering / Sort Key Recommendations

**BigQuery:**
```sql
ALTER TABLE fct_orders_placed
SET OPTIONS (
  require_partition_filter = TRUE
);
-- Cluster on columns used in WHERE and GROUP BY after the partition filter:
CREATE OR REPLACE TABLE fct_orders_placed
PARTITION BY DATE(order_date)
CLUSTER BY customer_sk, product_sk
AS SELECT * FROM fct_orders_placed;
```

**Snowflake:**
```sql
ALTER TABLE fct_orders_placed
CLUSTER BY (order_date, customer_sk);
-- Note: Snowflake clustering is a background maintenance operation;
-- it re-orders micro-partitions over time, not immediately.
```

**Amazon Redshift:**
```sql
-- DISTKEY: column used in JOIN predicates (most common join column)
-- SORTKEY: column(s) used in WHERE range filters (typically date)
CREATE TABLE fct_orders_placed (
  customer_sk   BIGINT NOT NULL DISTKEY,
  order_date    DATE   NOT NULL,
  ...
)
SORTKEY (order_date, customer_sk)
DISTSTYLE KEY;
```

**PostgreSQL:** Use B-tree indexes for filter columns; `CLUSTER` command to physically reorder the table:
```sql
CREATE INDEX idx_orders_date_customer ON fct_orders_placed (order_date, customer_sk);
CLUSTER fct_orders_placed USING idx_orders_date_customer;
```

## Step 7: Materialization Recommendations

Identify views or query patterns that should be pre-computed:

| Object | Type | Refresh Strategy | Estimated Savings | DDL |
|---|---|---|---|---|
| `daily_sales_by_region` | Materialized View | Incremental daily | Reduces dashboard load from 30s to <1s | See below |

**BigQuery:**
```sql
CREATE MATERIALIZED VIEW daily_sales_by_region AS
SELECT
  DATE(order_date) AS report_date,
  c.region,
  SUM(f.revenue_usd) AS total_revenue,
  COUNT(DISTINCT f.customer_sk) AS unique_customers
FROM fct_orders_placed f
JOIN dim_customer c ON f.customer_sk = c.customer_sk
GROUP BY 1, 2;
```

**Snowflake:**
```sql
CREATE DYNAMIC TABLE daily_sales_by_region
  TARGET_LAG = '1 hour'
  WAREHOUSE = transforming_wh
AS
SELECT ...;
```

**PostgreSQL:**
```sql
CREATE MATERIALIZED VIEW daily_sales_by_region AS
SELECT ...;
CREATE UNIQUE INDEX ON daily_sales_by_region (report_date, region);
-- Refresh with: REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_by_region;
```

## Step 8: Denormalization Opportunities

Identify dimension attributes that are joined into nearly every query and should be embedded in the fact table or a pre-joined wide table:

| Attribute | Current Location | Recommended Move | Trade-Off |
|---|---|---|---|
| `customer.region` | `dim_customer` | Embed in `fct_orders_placed` as `customer_region` | +Storage; -Join cost on all region-filtered reports |
| `product.category_name` | `dim_product` → `dim_category` | Embed in `dim_product` (partial denormalization) | +Query simplicity; -Update anomaly if category changes |

For each recommendation: provide the `ALTER TABLE ADD COLUMN` statement and the `UPDATE` statement to backfill it.

## Wrap-Up: Implementation Priority Matrix

Present all recommendations sorted by impact × effort:

| # | Recommendation | Type | Platform | Impact | Effort | Priority |
|---|---|---|---|---|---|---|
| 1 | Partition `fct_orders_placed` by `order_date` | Partitioning | All | HIGH | LOW | **P1** |
| 2 | Cluster `fct_orders_placed` by `customer_sk` | Clustering | BigQuery/Snowflake | HIGH | LOW | **P1** |
| 3 | Materialise `daily_sales_by_region` | Materialisation | All | HIGH | MEDIUM | **P2** |
| 4 | Add B-tree index on `dim_customer.region` | Index | PostgreSQL only | MEDIUM | LOW | **P2** |
| 5 | Embed `customer_region` in fact table | Denormalization | All | MEDIUM | MEDIUM | **P3** |
| 6 | Add DISTKEY on `customer_sk` | Distribution | Redshift only | HIGH | MEDIUM | **P2** |

**Priority definitions:**
- **P1** — Implement immediately; high impact, low effort
- **P2** — Implement in next sprint; high impact but requires planning or testing
- **P3** — Consider when other improvements are exhausted; lower impact or high effort

---

## Self-Check Before Finalising

- [ ] Every recommendation is tied to a specific table, column, or query — no generic advice
- [ ] Platform is explicitly stated for every recommendation (especially index types)
- [ ] Quantified benefit estimates are provided where possible ("reduces scan from 100% to ~3%")
- [ ] Redshift recommendations use DISTKEY/SORTKEY correctly (not traditional indexes)
- [ ] BigQuery/Snowflake recommendations do not mention B-tree indexes
- [ ] Priority matrix is sorted correctly (P1 = high impact + low effort first)
- [ ] Denormalization recommendations include the trade-off (storage vs. update anomaly risk)
