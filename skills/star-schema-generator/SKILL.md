---
name: star-schema-generator
description: Generates a complete Kimball-style star schema from business requirements or entity descriptions. Use when the user provides a business process, domain entities, or reporting requirements and wants a fact table, dimension tables, SQL DDL, and grain definition.
---

# Star Schema Generator

## Step 1: Load Shared Resources

Before proceeding, read the following files from this repository:
- `shared/glossary.md` — canonical definitions for all data modelling terms
- `shared/naming-conventions.md` — naming standards for all generated tables and columns

## Step 2: Gather Requirements

If the user has not provided all of the following, ask for them before proceeding. Do not guess.

**Required inputs:**
1. **Business process** — What event or activity is being measured? (e.g. "customer purchases", "patient admissions", "web sessions")
2. **Entities involved** — What are the key objects in this process? (e.g. Customer, Product, Store, Date, Promotion)
3. **Desired grain** — What does one row represent? If not stated, propose a grain and ask the user to confirm.
4. **Key metrics** — What measures should be captured? (e.g. revenue, quantity, discount, duration)
5. **SQL dialect** — BigQuery (default), Snowflake, Redshift, or PostgreSQL

**Optional but valuable:**
- Existing table structures or column lists
- Known reporting requirements (e.g. "we need to report by region and month")
- Any known many-to-many relationships

## Step 3: Define the Grain

Write a single-sentence grain statement in this exact form:

> "Each row in `fct_<process>` represents one `<grain unit>` for `<subject>` at the level of `<lowest meaningful unit>`."

Example: "Each row in `fct_orders_placed` represents one order line item for one product on one order."

**Do not proceed past this step until the grain is confirmed.** If the grain is ambiguous, present two or three candidate grain options and ask the user to choose.

## Step 4: Identify Dimension Tables

For each entity that provides descriptive context for the fact, define a dimension table:

| Dimension Table | Business Key | Key Descriptive Attributes | Notes |
|---|---|---|---|
| `dim_<entity>` | `<entity>_bk` | list attributes | conformed / role-playing / junk |

Flag any of the following:
- **Conformed dimensions** — dimensions shared with other fact tables
- **Role-playing dimensions** — the same physical table used in multiple FK roles (e.g. `dim_date` as `order_date_sk`, `ship_date_sk`)
- **Junk dimension candidates** — groups of low-cardinality flags that should be bundled (e.g. `is_promotion`, `is_online`, `payment_method`)
- **Degenerate dimensions** — keys with no descriptive attributes that should live directly in the fact table (e.g. `order_number`)

Always include a `dim_date` dimension unless the user explicitly says there is no time dimension.

## Step 5: Design the Fact Table

Define the fact table with:

**Foreign keys** (one per dimension, using surrogate keys):
- `<dim>_sk` referencing `dim_<entity>(<dim>_sk)`
- Role-playing FKs named `<role>_<dim>_sk` (e.g. `ship_date_sk`, `order_date_sk`)

**Degenerate dimensions** (named `<source_key>_ddk`):
- Source system identifiers that belong logically in the fact row

**Measures** (classify each one):
- **Fully additive** — can be summed across all dimensions (e.g. `revenue_usd`, `quantity_units`)
- **Semi-additive** — can be summed across some dimensions but not time (e.g. `account_balance_usd` — note correct aggregation: point-in-time, not SUM across dates)
- **Non-additive** — cannot be summed (e.g. ratios, percentages — document the formula instead of storing the computed ratio)

## Step 6: Generate SQL DDL

Generate `CREATE TABLE` statements in the following order:
1. All dimension tables (each with surrogate key as PK, business key with `UNIQUE` constraint, all descriptive columns)
2. The fact table (all FK columns referencing dimension PKs, all measure columns, all degenerate dimension columns)

Use naming conventions from `shared/naming-conventions.md` and dialect syntax from `shared/sql-dialect-guide.md`.

Example dimension DDL pattern (BigQuery):
```sql
CREATE TABLE IF NOT EXISTS dim_customer (
  customer_sk     INT64       NOT NULL,
  customer_bk     STRING      NOT NULL,
  customer_name   STRING,
  email           STRING,
  region          STRING,
  effective_date  DATE,
  expiry_date     DATE,
  is_current      BOOL        DEFAULT TRUE,
  _loaded_at      TIMESTAMP   DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (customer_sk) NOT ENFORCED,
  UNIQUE KEY (customer_bk) NOT ENFORCED
);
```

Example fact DDL pattern (BigQuery):
```sql
CREATE TABLE IF NOT EXISTS fct_orders_placed (
  order_line_sk     INT64     NOT NULL,
  customer_sk       INT64     NOT NULL,
  product_sk        INT64     NOT NULL,
  order_date_sk     INT64     NOT NULL,
  ship_date_sk      INT64,
  store_sk          INT64,
  order_number_ddk  STRING,
  revenue_usd       NUMERIC(18,2),
  quantity_units    INT64,
  discount_amount_usd NUMERIC(18,2),
  _loaded_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (order_line_sk) NOT ENFORCED,
  FOREIGN KEY (customer_sk)   REFERENCES dim_customer(customer_sk)   NOT ENFORCED,
  FOREIGN KEY (product_sk)    REFERENCES dim_product(product_sk)     NOT ENFORCED,
  FOREIGN KEY (order_date_sk) REFERENCES dim_date(date_sk)           NOT ENFORCED,
  FOREIGN KEY (ship_date_sk)  REFERENCES dim_date(date_sk)           NOT ENFORCED
);
```

## Step 7: Generate dbt-Style Documentation

Produce a `schema.yml` documentation block for all tables:

```yaml
version: 2

models:
  - name: dim_customer
    description: "One row per current and historical version of a customer."
    columns:
      - name: customer_sk
        description: "Surrogate key for the customer dimension."
        tests: [not_null, unique]
      - name: customer_bk
        description: "Business key from the source CRM system."
        tests: [not_null]
      ...

  - name: fct_orders_placed
    description: "One row per order line item. Grain: one product on one order."
    columns:
      - name: order_line_sk
        description: "Surrogate key for the order line fact."
        tests: [not_null, unique]
      - name: revenue_usd
        description: "Gross revenue in USD. Fully additive across all dimensions."
      ...
```

## Step 8: Flag Issues and Recommendations

Review the design and flag any of the following:

| Issue | Severity | Recommendation |
|---|---|---|
| Many-to-many relationship found | HIGH | Add a bridge table `brd_<fact>_<dim>` |
| No date dimension included | MEDIUM | Add `dim_date` and a `<event>_date_sk` FK |
| Non-additive measure stored as raw ratio | MEDIUM | Store numerator + denominator; compute ratio in the BI layer |
| Role-playing dimension not aliased | LOW | Create views `dim_order_date`, `dim_ship_date` aliasing `dim_date` |
| Composite natural key used as fact FK | HIGH | Generate a surrogate key for the dimension; use it as the FK |

## Wrap-Up Output Format

Produce the final output in this exact structure:

---

## Star Schema Design: `<Business Process>`

### Grain Statement
> Each row in `fct_<process>` represents one `<grain unit>` ...

### Dimension Tables
| Table | Business Key | Key Attributes | Notes |
|---|---|---|---|

### Fact Table
| Column | Type | Role | Additivity |
|---|---|---|---|

### SQL DDL
```sql
-- Dimension tables first, then fact table
```

### Documentation (dbt schema.yml)
```yaml
version: 2
models:
  ...
```

### Issues & Recommendations
| Issue | Severity | Recommendation |
|---|---|---|

---

## Self-Check Before Finalising

Before outputting the final design, verify:
- [ ] Grain statement is a single sentence in the prescribed form
- [ ] Every dimension has a surrogate key as PK and a business key with UNIQUE constraint
- [ ] Fact table contains only FKs, measures, and degenerate dimensions — no descriptive attributes
- [ ] Every FK in the fact table references the correct dimension PK
- [ ] All measures are classified by additivity type
- [ ] Naming follows `shared/naming-conventions.md` throughout
- [ ] DDL uses the correct dialect syntax from `shared/sql-dialect-guide.md`
- [ ] All issues and flags are documented
