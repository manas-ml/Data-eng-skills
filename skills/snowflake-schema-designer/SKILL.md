---
name: snowflake-schema-designer
description: Designs a normalized snowflake schema from hierarchical or multi-level dimensional data. Use when dimensions have sub-dimensions (e.g. Product → Subcategory → Category → Department), when the user explicitly wants 3NF normalization within dimensions, or when storage efficiency and hierarchy browsing are priorities over query simplicity.
---

# Snowflake Schema Designer

> **Important clarification:** "Snowflake schema" refers to a dimensional modelling pattern where dimension tables are normalized into hierarchies. It is unrelated to Snowflake Inc., the cloud data warehouse product.

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md`
- `shared/naming-conventions.md`
- `shared/sql-dialect-guide.md`

## Step 2: Gather Requirements

If the user has not provided the following, ask before proceeding:

1. **Parent dimension(s) to normalize** — Which dimension tables have hierarchical attributes? (e.g. "Product has: product → subcategory → category → department")
2. **Attributes per level** — What descriptive columns belong at each hierarchy level?
3. **Cardinality estimates** (optional but recommended) — How many distinct values at each level? This informs whether normalization provides meaningful storage savings.
4. **Reason for normalization** — Storage savings? Update simplicity? Hierarchy browsing in BI tools? This affects the recommendation.
5. **SQL dialect** — BigQuery (default), Snowflake, Redshift, or PostgreSQL
6. **Fact table context** — What fact table(s) reference the top-level dimension?

## Step 3: Decompose Dimension Hierarchies

For each parent dimension, decompose into a chain of sub-dimension tables:

**Hierarchy chain (example):**
```
fct_orders_placed
  └── dim_product (product_sk PK, subcategory_sk FK)
        └── dim_subcategory (subcategory_sk PK, category_sk FK)
              └── dim_category (category_sk PK, department_sk FK)
                    └── dim_department (department_sk PK)
```

For each level, document:
| Level | Table Name | PK | FK to parent | Level-Specific Attributes |
|---|---|---|---|---|
| Leaf | `dim_product` | `product_sk` | `subcategory_sk` | product_name, sku, price, weight |
| 2nd | `dim_subcategory` | `subcategory_sk` | `category_sk` | subcategory_name, subcategory_code |
| 3rd | `dim_category` | `category_sk` | `department_sk` | category_name, category_manager |
| Root | `dim_department` | `department_sk` | — | department_name, region |

**Rule:** Attributes belong at the level where they are functionally determined. `category_manager` belongs to `dim_category`, not `dim_product`.

## Step 4: Produce the ERD Chain Outline

Present a visual hierarchy outline before generating DDL:

```
fct_orders_placed
  ├── dim_customer     (no sub-dimensions — stays as star)
  ├── dim_date         (no sub-dimensions — stays as star)
  └── dim_product      [SNOWFLAKED]
        ├── product_name, sku, price
        └── → dim_subcategory
              ├── subcategory_name
              └── → dim_category
                    ├── category_name, category_manager
                    └── → dim_department
                          └── department_name, region
```

Label dimensions that remain flat (star) vs. those that are snowflaked.

## Step 5: Generate SQL DDL (Bottom-Up Order)

Generate DDL starting from the deepest (root) level and working up to the leaf. This ensures FK references are valid.

Order: root → … → leaf → (fact table FK column update)

Example (BigQuery):
```sql
-- Level 4: Root
CREATE TABLE IF NOT EXISTS dim_department (
  department_sk   INT64  NOT NULL,
  department_bk   STRING NOT NULL,
  department_name STRING,
  region          STRING,
  _loaded_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (department_sk) NOT ENFORCED,
  UNIQUE KEY (department_bk) NOT ENFORCED
);

-- Level 3
CREATE TABLE IF NOT EXISTS dim_category (
  category_sk       INT64  NOT NULL,
  category_bk       STRING NOT NULL,
  category_name     STRING,
  category_manager  STRING,
  department_sk     INT64  NOT NULL,
  _loaded_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (category_sk) NOT ENFORCED,
  FOREIGN KEY (department_sk) REFERENCES dim_department(department_sk) NOT ENFORCED
);

-- Level 2
CREATE TABLE IF NOT EXISTS dim_subcategory (
  subcategory_sk   INT64  NOT NULL,
  subcategory_bk   STRING NOT NULL,
  subcategory_name STRING,
  subcategory_code STRING,
  category_sk      INT64  NOT NULL,
  _loaded_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (subcategory_sk) NOT ENFORCED,
  FOREIGN KEY (category_sk) REFERENCES dim_category(category_sk) NOT ENFORCED
);

-- Level 1: Leaf
CREATE TABLE IF NOT EXISTS dim_product (
  product_sk       INT64         NOT NULL,
  product_bk       STRING        NOT NULL,
  product_name     STRING,
  sku              STRING,
  unit_price       NUMERIC(18,2),
  weight_kg        NUMERIC(10,3),
  subcategory_sk   INT64         NOT NULL,
  _loaded_at       TIMESTAMP     DEFAULT CURRENT_TIMESTAMP(),
  PRIMARY KEY (product_sk) NOT ENFORCED,
  FOREIGN KEY (subcategory_sk) REFERENCES dim_subcategory(subcategory_sk) NOT ENFORCED
);
```

## Step 6: Mandatory Trade-Off Analysis

This section is required in every snowflake schema design output.

Compare the snowflake design to an equivalent star schema for the same data:

| Dimension | Star Schema | Snowflake Schema |
|---|---|---|
| `dim_product` rows | e.g. 50,000 | 50,000 |
| Total dimension rows | 50,000 | 50,000 + 500 + 80 + 12 = 50,592 |
| Storage saving | Baseline | ~Marginal unless category attributes are very wide |
| Query joins needed | 1 | 4 (product → subcategory → category → department) |
| Update simplicity | Update all product rows when category changes | Update only `dim_category` rows |
| BI tool compatibility | Excellent (single table) | Reduced (multi-hop joins may confuse some tools) |

**Recommendation:** State whether the snowflake design is justified for this specific case. Snowflake schemas are most beneficial when:
- Upper hierarchy levels have many wide attributes that repeat millions of times
- Hierarchy levels change independently (e.g. product re-categorisation)
- The BI tool natively supports hierarchy browsing across FK chains

## Step 7: Flattened Materialized View

Always provide a flattened view that re-creates the star-schema equivalent for BI tools or users who cannot traverse multi-hop joins:

```sql
CREATE VIEW dim_product_flat AS
SELECT
  p.product_sk,
  p.product_bk,
  p.product_name,
  p.sku,
  p.unit_price,
  p.weight_kg,
  s.subcategory_sk,
  s.subcategory_name,
  s.subcategory_code,
  c.category_sk,
  c.category_name,
  c.category_manager,
  d.department_sk,
  d.department_name,
  d.region
FROM dim_product       p
JOIN dim_subcategory   s ON p.subcategory_sk = s.subcategory_sk
JOIN dim_category      c ON s.category_sk    = c.category_sk
JOIN dim_department    d ON c.department_sk   = d.department_sk;
```

Note: For BigQuery and Snowflake, consider materializing this as a table rather than a view if the hierarchy is queried frequently.

## Step 8: Generate dbt Documentation

Produce `schema.yml` entries for all sub-dimension tables:

```yaml
version: 2
models:
  - name: dim_department
    description: "Root hierarchy level for product classification."
    columns:
      - name: department_sk
        tests: [not_null, unique]
      - name: department_bk
        tests: [not_null, unique]
  - name: dim_product_flat
    description: "Flattened view of the product snowflake hierarchy for BI tool compatibility."
```

## Wrap-Up Output Format

---

## Snowflake Schema Design: `<Dimension Name>` Hierarchy

### Hierarchy Chain
```
fct_... → dim_... → dim_... → dim_... → dim_...
```

### Level Decomposition Table
| Level | Table | PK | FK | Attributes |
|---|---|---|---|---|

### SQL DDL (bottom-up)
```sql
...
```

### Flattened View
```sql
CREATE VIEW dim_<entity>_flat AS ...
```

### Trade-Off Analysis
| Factor | Star Schema | Snowflake Schema |
|---|---|---|

### Recommendation
[Justified recommendation for or against snowflaking in this specific case]

### dbt Documentation
```yaml
...
```

---

## Self-Check Before Finalising

- [ ] FK chain is complete — every level references the level above it
- [ ] Attributes are placed at the correct level (determined by where they are functionally dependent)
- [ ] DDL is generated bottom-up (root first)
- [ ] Flattened view joins all levels and includes all leaf + parent attributes
- [ ] Trade-off analysis compares row counts, storage, join count, and BI compatibility
- [ ] Recommendation states whether snowflaking is justified for this case
