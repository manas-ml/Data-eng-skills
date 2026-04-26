---
name: grain-definer
description: Helps define and validate the grain of a fact table. Identifies conformed dimensions, checks for fanout and chasm traps in join paths, and validates whether measures are additive at the declared grain. Use when the user is designing a new fact table, is unsure of the correct grain, or when a fact table is producing incorrect aggregation results (inflated totals, missing rows, wrong measures).
---

# Grain Definer

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md` — definitions for Grain, Fanout Trap, Chasm Trap, Conformed Dimension, Semi-additive, Degenerate Dimension
- `shared/naming-conventions.md`

## Step 2: Gather Inputs

Ask for any inputs not provided:

**Required:**
1. **Business process being measured** — What event or activity does this fact table record? (e.g. "customer purchases", "inventory snapshots", "web page views", "insurance claims")
2. **Proposed or existing fact table** — DDL, column list, or description
3. **Dimension tables involved** — List of dimensions that join to this fact table (names and FK columns)

**Strongly recommended:**
4. **Sample queries producing unexpected results** — If the user is debugging inflated totals or missing rows, the problematic SQL is the most valuable input for trap detection
5. **Expected vs actual result** — "I expect 1,000 total orders but my query returns 4,200"

**Optional:**
6. **Source system grain** — "Each row in the source represents one invoice line" — this often directly determines the correct grain

## Step 3: Define the Grain Statement

Guide the user to complete the grain statement in this exact form:

> "Each row in `fct_<process>` represents one `<event type>` of `<subject>` at the level of `<lowest meaningful unit>`."

**Examples of well-defined grain statements:**
- "Each row in `fct_orders_placed` represents one order line item (one product on one order)."
- "Each row in `fct_inventory_snapshot` represents the stock level of one product at one warehouse at the end of one day."
- "Each row in `fct_web_sessions` represents one user session on one page."
- "Each row in `fct_claims_submitted` represents one insurance claim line for one procedure code on one claim."

**If the grain is ambiguous**, present two or three candidate options and ask the user to choose:

> "Based on your description, there are two possible grains. Please confirm which is correct:
> - **Option A (Order level):** One row per order → `order_id` is unique in the fact table
> - **Option B (Order line level):** One row per order line → `(order_id, line_number)` is unique
>
> The correct grain affects which dimensions are valid and which measures are additive."

**Do not proceed past this step until the grain is confirmed.**

## Step 4: Validate the Grain Against the Schema

Once the grain is confirmed, validate that the fact table structure supports it:

1. **Uniqueness check** — Is there a PK or UNIQUE constraint on the grain column(s)?
2. **Granularity consistency** — Are all FK dimensions consistent with the grain? Example: if grain is "order line", each FK should reference a dimension that applies at the order-line level, not the order header level.
3. **Lowest granularity available** — Is the stated grain the lowest level available in the source system? Aggregating too early (choosing a coarser grain) loses detail that cannot be recovered later.

Flag any inconsistency:

| Check | Result | Notes |
|---|---|---|
| Surrogate PK exists for grain | Pass / Fail | |
| All FKs appropriate for this grain | Pass / Fail | |
| Grain is at lowest available source granularity | Pass / Fail | |

## Step 5: Conformed Dimension Classification

For each dimension that joins to this fact table, classify it:

| Dimension | Type | Notes |
|---|---|---|
| `dim_date` | **Conformed** | Shared with `fct_inventory_snapshot` and other facts |
| `dim_customer` | **Conformed** | Shared with `fct_returns` |
| `dim_order_status` | **Junk dimension candidate** | Low-cardinality flag; consider bundling with other flags |
| `dim_date` (second join as `ship_date_sk`) | **Role-playing** | Same table used twice; create views `dim_order_date`, `dim_ship_date` |
| `dim_promotion` | **Potential M:M** | A single order line may have multiple promotions applied → investigate |

**Conformed dimension guidance:**
If a dimension is used across multiple fact tables, ensure it uses the same surrogate key and business key. Inconsistent conformed dimensions cause incorrect drill-across reports.

**Role-playing dimension guidance:**
When `dim_date` appears as multiple FK columns (`order_date_sk`, `ship_date_sk`, `delivery_date_sk`), create alias views:
```sql
CREATE VIEW dim_order_date AS SELECT * FROM dim_date;
CREATE VIEW dim_ship_date AS SELECT * FROM dim_date;
```

## Step 6: Fanout Trap Detection

A fanout trap occurs when joining a fact table to a dimension produces more rows than the fact table has, because one fact row matches multiple dimension rows (1:M relationship on the join path).

**This is the most common cause of inflated aggregation totals.**

### Diagnostic Query

Run this diagnostic for every dimension join that might cause a fanout:

```sql
-- Replace <fact_table>, <dim_table>, <grain_key>, <fact_fk>, <dim_pk> with actual names
SELECT
  COUNT(*)                          AS total_rows_after_join,
  COUNT(DISTINCT f.<grain_key>)     AS distinct_grain_keys_after_join
FROM <fact_table> f
JOIN <dim_table> d ON f.<fact_fk> = d.<dim_pk>
;

-- If total_rows_after_join > distinct_grain_keys_after_join → FANOUT TRAP EXISTS
```

**Example:**
```sql
-- Testing: does dim_promotion cause a fanout on fct_orders_placed?
SELECT
  COUNT(*)                              AS total_rows,
  COUNT(DISTINCT f.order_line_sk)       AS distinct_order_lines
FROM fct_orders_placed f
JOIN dim_promotion d ON f.promotion_sk = d.promotion_sk
;
-- Result: total_rows=4200, distinct_order_lines=1000 → FANOUT TRAP (4.2x inflation)
```

**Resolution for fanout traps:**
- If the dimension is truly M:M with the fact (e.g. multiple promotions per order line) → introduce a bridge table
- If the FK should have been unique → fix the data or the dimension design

```sql
-- Bridge table pattern for M:M promotions:
CREATE TABLE brd_orders_promotions (
  order_line_sk   INT64   NOT NULL,
  promotion_sk    INT64   NOT NULL,
  weighting_factor NUMERIC(5,4) DEFAULT 1.0,
  PRIMARY KEY (order_line_sk, promotion_sk) NOT ENFORCED
);
```

## Step 7: Chasm Trap Detection

A chasm trap occurs when two dimensions (or fact tables) that share a common entity are joined through that entity, and rows are lost because the entity has records in one table but not the other.

### Diagnostic Query

```sql
-- Chasm trap check: customers with orders but no payments
-- (Tests whether a shared entity has records in both related tables)
SELECT
  c.customer_sk,
  COUNT(o.order_sk)   AS order_count,
  COUNT(p.payment_sk) AS payment_count
FROM dim_customer c
LEFT JOIN fct_orders_placed o ON c.customer_sk = o.customer_sk
LEFT JOIN fct_payments p      ON c.customer_sk = p.customer_sk
GROUP BY c.customer_sk
HAVING COUNT(o.order_sk) > 0 AND COUNT(p.payment_sk) = 0
;
-- If rows returned → customers have orders but no payments → chasm trap in cross-fact queries
```

**Resolution for chasm traps:**
- Use separate queries per fact table and combine in the BI layer (outer joins)
- Never inner-join two fact tables via a shared dimension in a single query
- Use `FULL OUTER JOIN` when cross-fact analysis is necessary

## Step 8: Measure Additivity Validation

For each numeric measure column in the fact table, classify its additivity:

| Measure | Additivity Type | Correct Aggregation | Notes |
|---|---|---|---|
| `revenue_usd` | Fully additive | `SUM(revenue_usd)` | Correct to sum across all dimensions |
| `quantity_units` | Fully additive | `SUM(quantity_units)` | |
| `discount_amount_usd` | Fully additive | `SUM(discount_amount_usd)` | |
| `account_balance_usd` | **Semi-additive** | `AVG(account_balance_usd)` across time; `SUM` across accounts at a point in time | Cannot SUM across dates — gives wrong YTD |
| `discount_percentage` | **Non-additive** | Do not store — compute from `discount_amount / revenue` | Storing a ratio as a measure is an anti-pattern |
| `inventory_level` | **Semi-additive** | Point-in-time value; `SUM` across warehouses OK; NOT across dates | Use `/scd-designer` for snapshot handling |

**Semi-additive measure guidance** (flag prominently):
> ⚠️ `account_balance_usd` is semi-additive. It should NOT be summed across the date dimension. Use:
> - Point-in-time: `WHERE date_sk = <specific_date_sk>`
> - Period-end balance: `WHERE is_period_end = TRUE`
> - Average: `AVG(account_balance_usd)` across date range

**Non-additive measure guidance:**
> ⚠️ `discount_percentage` is non-additive and should not be stored as a measure. Instead, store:
> - `discount_amount_usd` (numerator) — fully additive
> - `revenue_usd` (denominator) — fully additive
> - Compute `discount_percentage = SUM(discount_amount) / SUM(revenue)` in the BI layer

## Step 9: Grain Validation Query

Provide a SQL query the user can run to confirm the grain is correctly implemented:

```sql
-- Grain validation: COUNT(*) should equal COUNT(DISTINCT grain_key)
-- Replace <fact_table> and <grain_key> with actual names
SELECT
  COUNT(*)                      AS total_rows,
  COUNT(DISTINCT <grain_key>)   AS distinct_grain_keys,
  COUNT(*) - COUNT(DISTINCT <grain_key>) AS duplicates
FROM <fact_table>
;
-- Expected: duplicates = 0
```

## Wrap-Up Output Format

---

## Grain Definition Report: `<Business Process>`

### Confirmed Grain Statement
> "Each row in `fct_<process>` represents one `<unit>` of `<subject>`."

### Grain Validation
| Check | Status | Notes |
|---|---|---|

### Dimension Classification
| Dimension | FK Column | Type | Action Required |
|---|---|---|---|

### Fanout Trap Analysis
For each dimension tested:
| Dimension | Diagnostic Result | Trap Detected? | Resolution |
|---|---|---|---|

### Chasm Trap Analysis
| Entity | Fact Table 1 | Fact Table 2 | Trap Detected? | Resolution |
|---|---|---|---|

### Measure Additivity
| Measure | Type | Correct Aggregation | Warning |
|---|---|---|---|

### Grain Validation SQL
```sql
SELECT COUNT(*), COUNT(DISTINCT <grain_key>), COUNT(*) - COUNT(DISTINCT <grain_key>) AS duplicates
FROM <fact_table>;
-- Expected: duplicates = 0
```

---

## Self-Check Before Finalising

- [ ] Grain statement is exactly one sentence in the prescribed form and is confirmed by the user
- [ ] Fanout trap diagnostic SQL is provided for every dimension that could cause inflation
- [ ] Chasm trap analysis is run proactively, even if the user did not ask for it
- [ ] Every semi-additive measure is flagged with a ⚠️ warning and the correct aggregation pattern
- [ ] Every non-additive measure (ratio, percentage) is flagged and the alternative (store numerator + denominator) is recommended
- [ ] Grain validation SQL is provided and expected result (0 duplicates) is stated
- [ ] Role-playing dimensions are identified and alias views are recommended
