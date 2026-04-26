# Data Modelling Glossary

Canonical definitions used across all skills in this library. When a term appears in a skill's instructions, refer here for the authoritative definition.

---

## Core Dimensional Modelling

**Grain**
The precise definition of what one row in a fact table represents. The grain must be declared as a single sentence before any fact table design proceeds. Example: "Each row represents one order line item for one product on one order." The grain determines which dimensions are valid and which measures are additive.

**Fact Table**
A table in a dimensional model that stores quantitative measurements (measures) of a business process. A fact table's rows are defined by its grain. Fact tables contain foreign keys to dimension tables and numeric measures. They are typically wide, narrow in cardinality descriptors, and very large in row count.

**Dimension Table**
A table that provides the descriptive context (who, what, where, when, why, how) for the measurements stored in a fact table. Dimension tables are typically denormalized, wide in columns, and smaller in row count than fact tables.

**Conformed Dimension**
A dimension table that is shared identically (or as a subset) across two or more fact tables in the same data warehouse. Conformed dimensions enable consistent drill-across reporting between separate fact tables. Example: `dim_date` shared between `fct_sales` and `fct_inventory`.

**Degenerate Dimension**
A dimension attribute stored directly in the fact table rather than in a separate dimension table, because it has no descriptive attributes beyond its key. Example: order number stored in `fct_order_lines` — it is a dimension conceptually (it identifies the parent order) but carries no descriptive columns of its own.

**Role-Playing Dimension**
A single physical dimension table used multiple times in the same fact table, each time in a different role. Example: `dim_date` used as `order_date_sk`, `ship_date_sk`, and `delivery_date_sk` in a single fact table. Typically implemented using database views or aliases, not physical copies.

**Junk Dimension**
A dimension table that consolidates multiple low-cardinality flags and indicators from the fact table into a single surrogate-keyed table. Example: combining `is_promotion`, `is_online`, `payment_method` into a single `dim_transaction_flags` table to reduce fact table width.

**Bridge Table**
A table that resolves a many-to-many relationship between a fact table and a dimension that has multiple values per fact row. Example: a fact row for a customer account that has multiple advisors. The bridge table holds (account_sk, advisor_sk) pairs with optional weighting factors.

---

## Keys

**Surrogate Key**
A system-generated integer or UUID used as the primary key of a dimension table. Has no business meaning. Enables SCD Type 2 history by allowing multiple rows for the same business entity with different surrogate keys. Named `<table>_sk` by convention (e.g. `customer_sk`).

**Business Key (Natural Key)**
The identifier used by the source system to uniquely identify an entity. Has business meaning. Examples: customer email, product SKU, order number. Used in dimension lookups but not as the primary key in dimensional models. Named `<entity>_bk` by convention (e.g. `customer_bk`).

**Natural Key**
Synonym for Business Key. An identifier that comes from the source system and has meaning outside the data warehouse.

**Composite Key**
A primary key made up of two or more columns. Common in fact tables (e.g. PK = `order_sk + product_sk + date_sk`) and in SCD Type 2 satellite tables (e.g. PK = `customer_sk + effective_date`).

---

## Slowly Changing Dimensions (SCD)

**SCD Type 1**
Overwrite the old value with the new value. No history is kept. Used when history of the changed attribute is irrelevant. Example: correcting a spelling error in a customer name.

**SCD Type 2**
Add a new row for each change, with `effective_date`, `expiry_date`, and `is_current` columns to track the full history. The surrogate key changes with each version. Used when full historical point-in-time accuracy is required. Example: tracking customer address changes for historical order attribution.

**SCD Type 3**
Add a new column to store the previous value alongside the current value. Only one level of history is preserved. Used when only the "before and after" of the most recent change is needed. Example: `current_region` and `previous_region` columns.

**SCD Type 4**
Split rapidly changing attributes into a separate "mini-dimension" table, keeping only a current FK in the main dimension. Used for high-cardinality or high-velocity attributes that would cause excessive Type 2 row proliferation. Example: separating customer income band and age group into a `dim_customer_profile` mini-dimension.

**SCD Type 6 (Hybrid)**
Combines Types 1 + 2 + 3. Uses Type 2 history rows, but also stamps the current value of tracked attributes on every historical row (Type 1 overwrite of a `current_<attr>` column). This allows queries to filter "current customers in segment X" without a subquery, while still preserving history. Often called "Type 6" because 1 + 2 + 3 = 6.

---

## Data Vault 2.0

**Hub**
A Data Vault table that stores the unique list of business keys for one business entity. Contains: hash key (PK), load date, record source, and the business key column(s). Named `hub_<entity>`.

**Link**
A Data Vault table that stores the relationships between two or more business entities (Hubs). Contains: link hash key (PK, hash of all participant business keys), load date, record source, and FK hash keys to each participating Hub. Named `lnk_<entity1>_<entity2>`.

**Satellite**
A Data Vault table that stores the descriptive attributes and change history for a Hub or Link. Contains: parent hash key (FK), load date (part of PK), load end date, record source, hash diff (hash of all descriptive attributes), and the descriptive columns. Named `sat_<entity>_<source_system>`.

**Hash Key**
A cryptographic hash (MD5 or SHA-256) of the business key(s), used as the surrogate key throughout the Data Vault. Must be computed consistently: `MD5(UPPER(TRIM(business_key)))`. NULL business keys use a sentinel value (e.g. `'^^'`). Composite business keys concatenate values with a separator (e.g. `'~'`).

**Hash Diff**
A hash of all descriptive attribute values in a Satellite row, used to detect changes efficiently without comparing every column individually. If `hash_diff` for the incoming row differs from the latest Satellite row for the same Hub key, a new Satellite record is inserted.

**Point-In-Time (PIT) Table**
A Data Vault helper table that records the latest Satellite load date for a given Hub key at each point in time. Enables efficient snapshot queries without full Satellite scans.

**Raw Vault**
The source-aligned layer of a Data Vault. Loads data as-is from source systems with minimal transformation. Contains Hubs, Links, and Satellites per source.

**Business Vault**
The business-rule layer above the Raw Vault. Contains derived or integrated Satellites, computed Links, and business-rule-applied entities.

---

## Normal Forms

**1NF (First Normal Form)**
A table is in 1NF if: (1) every column has atomic (indivisible) values — no comma-separated lists or repeating groups, (2) every column has a defined domain, (3) there are no duplicate rows.

**2NF (Second Normal Form)**
A table is in 2NF if it is in 1NF and every non-key column is fully functionally dependent on the entire primary key. Violations only exist when the primary key is composite — a non-key column that depends on only part of the PK violates 2NF.

**3NF (Third Normal Form)**
A table is in 3NF if it is in 2NF and there are no transitive functional dependencies: every non-key column depends only on the primary key, not on another non-key column. Mnemonic: "Every non-key attribute must depend on the key, the whole key, and nothing but the key."

**BCNF (Boyce-Codd Normal Form)**
A stricter form of 3NF. A table is in BCNF if every determinant (any column or set of columns that functionally determines another) is a candidate key. Violations arise when a non-key column determines part of a composite key.

---

## Join Traps

**Fanout Trap**
A query anomaly where joining a fact table to a dimension produces inflated row counts because one fact row matches multiple dimension rows (1:M relationship on the join). Causes SUM(measure) to be overcounted. Diagnostic: `SELECT COUNT(*), COUNT(DISTINCT grain_key) FROM fact JOIN dim ON ...` — if these differ, a fanout trap exists.

**Chasm Trap**
A query anomaly in a model where two fact tables share a common entity but are joined via that entity, causing rows to be lost when the entity has no matching rows in one of the fact tables. Classic example: joining `fct_orders` and `fct_payments` both through `dim_customer` when some customers have orders but no payments.

---

## dbt Concepts

**Source**
In dbt, a reference to a raw table loaded by an external process. Declared in `sources.yml`. Referenced in models using `{{ source('source_name', 'table_name') }}`.

**Staging Model**
A dbt model that sits directly on top of a source table. Renames columns, recasts data types, adds surrogate keys and metadata columns. No business logic. 1:1 with source tables. Materialised as views.

**Intermediate Model**
A dbt model that joins and reshapes staging models into business-entity-centric structures. Encapsulates business logic that is reused across multiple mart models. Optionally materialised as ephemeral or views.

**Mart Model**
A dbt model that produces a final, business-facing table — typically a fact table or a wide dimension. Materialised as a table or incrementally. Consumed directly by BI tools.

**Ref**
The dbt function `{{ ref('model_name') }}` used to reference other dbt models. Never hardcode schema or table names — always use `{{ ref() }}` for models and `{{ source() }}` for sources.

**Contract**
A dbt feature (v1.5+) that enforces column names and data types on a model's output. Declared in `schema.yml` with `contract: {enforced: true}`. Recommended for all mart-layer models.

---

## Modelling Philosophies

**Kimball (Dimensional Modelling)**
A bottom-up data warehouse design approach. Organises data into star or snowflake schemas optimised for analytical query performance and business user accessibility. Fact tables + conformed dimension tables. Denormalised dimensions for query simplicity.

**Inmon (Corporate Information Factory)**
A top-down data warehouse design approach. The enterprise data warehouse is built in 3NF first, then data marts are derived from it. Emphasises data integration and single source of truth before analytical optimisation.
