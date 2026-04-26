---
name: normalization-advisor
description: Analyzes a schema for compliance with normal forms (1NF through BCNF) and provides actionable normalization or deliberate denormalization recommendations with OLTP vs OLAP trade-off analysis. Use when the user provides a flat or partially normalized schema and wants guidance on whether and how to normalize it.
---

# Normalization Advisor

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md` — definitions for 1NF, 2NF, 3NF, BCNF, Kimball, Inmon
- `shared/naming-conventions.md`

## Step 2: Gather Inputs

Ask for any inputs not provided:

**Required:**
1. **Schema** — DDL (`CREATE TABLE` statements) or a table-column description in prose
2. **Primary use case** — OLTP (transactional, high write, normalized), OLAP (analytical, read-heavy, dimensional), or Mixed

**Optional:**
3. **Known functional dependencies** — If the user can describe which columns determine which other columns, the analysis will be more precise. Example: "zip_code determines city and state"
4. **Known access patterns** — Which queries are run most frequently? This informs the OLAP denormalization recommendations.

## Step 3: Parse and Tabulate the Schema

For each table, extract:
- Table name
- All columns with data types
- Declared primary key(s)
- Any UNIQUE constraints
- Any FK constraints

Present a summary table before analysis:

| Table | Columns | Declared PK | Notes |
|---|---|---|---|
| `customers` | id, name, email, city, state, zip, region | id | |
| `orders` | id, customer_id, order_date, status, total, tax, shipping | id | |

## Step 4: Test for First Normal Form (1NF)

For each table, check:

**1NF violations to look for:**
- **Multi-valued columns** — comma-separated lists stored in one column (e.g. `tags = 'sale,promo,clearance'`)
- **Repeating column groups** — multiple columns for the same attribute (e.g. `phone_1`, `phone_2`, `phone_3`)
- **Mixed types** — a column storing different types of values depending on context
- **No defined primary key** — duplicate rows are possible

For each violation found, document:

| Table | Column(s) | Violation Type | Description |
|---|---|---|---|
| `customers` | `phone_1`, `phone_2`, `phone_3` | Repeating group | Three phone columns instead of a separate `customer_phones` table |
| `orders` | `tags` | Multi-valued | Comma-separated values; not atomic |

**Fix recommendation for each 1NF violation:**
```sql
-- Fix repeating group: extract to a child table
CREATE TABLE customer_phones (
  phone_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customers(id),
  phone_type  VARCHAR(20),  -- 'mobile', 'home', 'work'
  phone_number VARCHAR(20) NOT NULL
);
ALTER TABLE customers DROP COLUMN phone_1, DROP COLUMN phone_2, DROP COLUMN phone_3;
```

## Step 5: Test for Second Normal Form (2NF)

**2NF only applies when the primary key is composite.** If all tables have single-column PKs, state: "2NF analysis is not applicable — all tables have single-column primary keys."

For tables with composite PKs, check: does every non-key column depend on the **entire** composite PK, or only part of it?

**Example violation:**
Table `order_lines (order_id, product_id, quantity, unit_price, product_name)` with PK `(order_id, product_id)`:
- `quantity` depends on the full composite PK ✓
- `unit_price` depends on the full composite PK ✓ (price at time of order)
- `product_name` depends only on `product_id` (partial dependency) ✗

| Table | Column | Partial Dependency On | Full PK |
|---|---|---|---|
| `order_lines` | `product_name` | `product_id` only | `(order_id, product_id)` |

**Fix:**
```sql
-- Move product_name to a products table
ALTER TABLE order_lines DROP COLUMN product_name;
-- product_name already exists in products table — join when needed
```

## Step 6: Test for Third Normal Form (3NF)

Check for transitive functional dependencies: A → B → C where A is the PK, B is a non-key column, and C is another non-key column that depends on B rather than directly on A.

Mnemonic: every non-key attribute must depend on **the key, the whole key, and nothing but the key**.

**Common 3NF violations:**
- `zip_code` → `city`, `zip_code` → `state` (transitive: zip determines city/state, not the PK)
- `employee_id` → `department_id` → `department_name` (department_name depends on department_id, not employee_id)
- `product_id` → `category_id` → `category_name`

| Table | PK | Transitive Dependency | Violating Column(s) |
|---|---|---|---|
| `customers` | `id` | `zip_code → city, state` | `city`, `state` |
| `employees` | `id` | `department_id → department_name` | `department_name` |

**Fix for each violation:**
```sql
-- Extract zip_code hierarchy to a reference table
CREATE TABLE zip_codes (
  zip_code  CHAR(5)      PRIMARY KEY,
  city      VARCHAR(100) NOT NULL,
  state     CHAR(2)      NOT NULL
);
ALTER TABLE customers DROP COLUMN city, DROP COLUMN state;
-- Now join customers → zip_codes to get city/state
```

## Step 7: Test for BCNF (Boyce-Codd Normal Form)

BCNF is stricter than 3NF. A table is in BCNF if every determinant is a candidate key.

BCNF violations arise in tables with **multiple overlapping candidate keys** where a non-key column determines part of one candidate key.

**Common BCNF violation pattern:**
- Table `course_teachers (student, course, teacher)` where: student + course → teacher, but also teacher → course. Here `teacher` determines `course` but `teacher` alone is not a candidate key.

Flag BCNF violations explicitly but note:
> **BCNF is rarely targeted in data warehousing.** For OLAP models, stop at 3NF or use star/snowflake design instead. BCNF decompositions can produce lossless-join but dependency-preserving issues.

## Step 8: OLTP vs OLAP Analysis

For each normalization recommendation, assess whether it is appropriate given the stated use case:

| Recommendation | 3NF Fix | OLTP Verdict | OLAP Verdict |
|---|---|---|---|
| Extract `city`, `state` from `customers` to `zip_codes` | Create `zip_codes` reference table | APPLY — eliminates update anomalies in transactional system | CONSIDER SKIPPING — adds a join to every region-filtered query; embed city/state in customer dim instead |
| Extract `product_name` from `order_lines` | Join to `products` table | APPLY — prevents product name duplication | ALREADY HANDLED — in dimensional model, product_name lives in `dim_product` |

**OLAP denormalization guidance:**
For OLAP use cases, deliberately NOT normalizing certain attributes (keeping `customer_region` in the orders table, embedding `product_category` in the product dimension) is the correct design. Recommend the Kimball star schema design instead of strict 3NF for OLAP workloads. Suggest using `/star-schema-generator` if the user wants an OLAP model.

## Step 9: Migration Risk Assessment

For each normalization change recommended, assess the migration impact:

| Change | Queries/Apps That Break | Migration SQL | Compatibility Strategy |
|---|---|---|---|
| Extract `city`/`state` to `zip_codes` | Any query `SELECT city FROM customers` | See below | Create view `customers_v` that joins back |
| Extract phone columns to `customer_phones` | Any query using `phone_1`, `phone_2` | See below | Create view with aggregated phones |

**Compatibility view pattern:**
```sql
-- Create a backward-compatible view so existing queries still work during migration
CREATE VIEW customers_v AS
SELECT
  c.id,
  c.name,
  c.email,
  c.zip_code,
  z.city,
  z.state
FROM customers c
JOIN zip_codes z ON c.zip_code = z.zip_code;
```

**Migration SQL:**
```sql
-- Step 1: Create new table
CREATE TABLE zip_codes (zip_code CHAR(5) PRIMARY KEY, city VARCHAR(100), state CHAR(2));

-- Step 2: Populate from existing data
INSERT INTO zip_codes
SELECT DISTINCT zip_code, city, state FROM customers;

-- Step 3: Drop old columns
ALTER TABLE customers DROP COLUMN city, DROP COLUMN state;

-- Step 4: Rename view to the original table name (after confirming all consumers are updated)
```

## Wrap-Up Output Format

---

## Normalization Analysis Report

### Model Overview
**Use Case:** `<OLTP / OLAP / Mixed>`
**Tables analysed:** `n`

### 1NF Results
| Table | Column | Violation | Fix |
|---|---|---|---|

### 2NF Results
`<Not applicable if no composite PKs>` or violation table

### 3NF Results
| Table | Transitive Dependency | Violating Columns | Fix |
|---|---|---|---|

### BCNF Results
`<Violations if any, or "No BCNF violations detected">`

### OLTP vs OLAP Recommendations
| Recommendation | Apply for OLTP? | Apply for OLAP? | Notes |
|---|---|---|---|

### Migration Risk Assessment
| Change | Impact | Migration SQL | Compatibility View |
|---|---|---|---|

---

## Self-Check Before Finalising

- [ ] Use case (OLTP/OLAP/Mixed) is stated before giving recommendations
- [ ] 2NF analysis states "not applicable" if no composite PKs exist
- [ ] Each violation cites the specific table, column(s), and the functional dependency at fault
- [ ] Every fix includes DDL (either ALTER TABLE or CREATE TABLE)
- [ ] OLAP denormalization guidance is included where relevant
- [ ] BCNF note is included (rarely targeted for OLAP)
- [ ] Migration risk assessment is provided for every recommended change
- [ ] Compatibility views are provided for breaking changes
- [ ] NEVER recommend BCNF for dimensional models; recommend star/snowflake instead
