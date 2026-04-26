---
name: dbt-model-builder
description: Converts raw SQL tables or source schemas into a full dbt project structure. Generates sources.yml, staging models, intermediate models, mart-layer models, and schema.yml with column-level tests. Use when the user provides raw table definitions or a source schema and wants a production-ready dbt layer.
---

# dbt Model Builder

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md`
- `shared/naming-conventions.md`

## Step 2: Gather Inputs

Ask for any inputs that are not provided:

**Required:**
1. **Source schema** — DDL (`CREATE TABLE` statements) or a list of source tables with columns
2. **Source system name** — Used in staging model file names (e.g. `salesforce`, `stripe`, `postgres_erp`)
3. **Desired mart output** — What business-facing model(s) does the user want? (e.g. "a `fct_orders_placed` fact table and `dim_customer` dimension")
4. **dbt adapter / platform** — BigQuery, Snowflake, Redshift, or PostgreSQL

**Optional:**
5. **Existing dbt project structure** — If a partial project exists, describe what's already there
6. **Business keys for each entity** — Which columns identify each entity uniquely in the source?
7. **Incremental load preference** — Should mart models be full-refresh or incremental?

## Step 3: Generate `sources.yml`

Create a `sources.yml` file that declares all raw source tables:

```yaml
# models/staging/<source_name>/sources.yml
version: 2

sources:
  - name: <source_name>
    database: "{{ env_var('DBT_DATABASE', 'your_database') }}"
    schema: raw_<source_name>
    description: "Raw tables loaded from <source_name>."
    loader: fivetran  # or airbyte, custom, etc.

    tables:
      - name: orders
        description: "Raw orders table from <source_name>."
        loaded_at_field: _fivetran_synced  # or updated_at, etl_loaded_at
        freshness:
          warn_after: {count: 6, period: hour}
          error_after: {count: 24, period: hour}
        columns:
          - name: id
            description: "Source system order identifier."
            tests: [not_null, unique]
          - name: customer_id
            description: "Source system customer identifier."
            tests: [not_null]
          - name: created_at
            description: "Timestamp when the order was created."

      - name: customers
        description: "Raw customers table from <source_name>."
        loaded_at_field: updated_at
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 48, period: hour}
        columns:
          - name: id
            tests: [not_null, unique]
```

**Rules:**
- One `sources.yml` per source system, placed in `models/staging/<source_name>/`
- Always declare `loaded_at_field` and `freshness` thresholds for production sources
- Always add `not_null` and `unique` tests to source PKs

## Step 4: Generate Staging Models

One staging model per source table. Staging models are 1:1 with source tables — no business logic.

**File naming:** `stg_<source>__<entity>.sql` (double underscore separates source from entity)

**Pattern for each staging model:**

```sql
-- models/staging/<source>/stg_<source>__<entity>.sql
{{
  config(
    materialized='view'
  )
}}

with source as (
    select * from {{ source('<source_name>', '<raw_table>') }}
),

renamed as (
    select
        -- Primary key: generate a surrogate key from the business key
        {{ dbt_utils.generate_surrogate_key(['id']) }} as <entity>_sk,

        -- Business key
        id                          as <entity>_bk,

        -- Renamed and recast columns
        customer_id                 as customer_bk,
        CAST(created_at AS TIMESTAMP) as order_created_at,
        CAST(updated_at AS TIMESTAMP) as order_updated_at,
        UPPER(TRIM(status))         as order_status,
        CAST(total_amount AS NUMERIC) as total_amount_usd,

        -- Metadata
        _fivetran_synced            as _loaded_at

    from source
    where _fivetran_deleted is false  -- exclude soft-deleted records if applicable
)

select * from renamed
```

**Rules for staging models:**
- ALWAYS use `{{ source() }}` to reference raw tables — never hardcode schema.table
- ALWAYS generate a surrogate key using `dbt_utils.generate_surrogate_key(['col1', 'col2'])`; list all business key columns explicitly
- Rename columns to use the warehouse naming convention; do not preserve source system names
- Recast data types here — ensure correct types for all downstream models
- Apply basic filters (e.g. exclude deleted records) but no business logic
- Materialized as `view`

## Step 5: Generate Intermediate Models

Intermediate models join and reshape staging models into business-entity-centric structures. They encapsulate logic reused across multiple mart models.

**File naming:** `int_<entity>_<transformation>.sql`

```sql
-- models/intermediate/int_orders_with_customer.sql
{{
  config(
    materialized='view'
  )
}}

with orders as (
    select * from {{ ref('stg_<source>__orders') }}
),

customers as (
    select * from {{ ref('stg_<source>__customers') }}
),

final as (
    select
        o.order_sk,
        o.order_bk,
        o.order_created_at,
        o.order_status,
        o.total_amount_usd,

        -- Customer context
        c.customer_sk,
        c.customer_bk,
        c.customer_name,
        c.region

    from orders o
    left join customers c on o.customer_bk = c.customer_bk
)

select * from final
```

**Rules:**
- ALWAYS use `{{ ref() }}` to reference staging or other dbt models
- One intermediate model per major business entity or join combination
- Business logic belongs here, not in staging
- Materialized as `view` or `ephemeral`; avoid materializing as table unless the join is very expensive

## Step 6: Generate Mart Models

Mart models are the final business-facing layer. Fact tables and dimension tables for BI consumption.

### Dimension Model Example

```sql
-- models/marts/dim_customer.sql
{{
  config(
    materialized='table',
    contract={
      'enforced': true
    }
  )
}}

with customers as (
    select * from {{ ref('stg_<source>__customers') }}
),

final as (
    select
        customer_sk,
        customer_bk,
        customer_name,
        email,
        region,
        country,
        created_at          as customer_since_at,
        _loaded_at
    from customers
)

select * from final
```

### Fact Model Example (Incremental)

```sql
-- models/marts/fct_orders_placed.sql
{{
  config(
    materialized='incremental',
    unique_key='order_sk',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
  )
}}

with orders as (
    select * from {{ ref('int_orders_with_customer') }}
    {% if is_incremental() %}
    where order_created_at >= (select max(order_created_at) from {{ this }})
    {% endif %}
),

final as (
    select
        order_sk,
        customer_sk,
        product_sk,
        order_date_sk,
        order_bk           as order_number_ddk,
        order_created_at,
        total_amount_usd   as revenue_usd,
        quantity_units,
        discount_amount_usd,
        _loaded_at
    from orders
)

select * from final
```

**Rules for mart models:**
- Fact tables: use `incremental` materialization with an explicit `unique_key`
- Dimension tables: use `table` materialization for full refresh; add `contract: {enforced: true}`
- ALWAYS use `{{ ref() }}` — never hardcode schema names
- Surrogate keys in mart models must be the final `*_sk` columns from staging; never recompute

## Step 7: Generate `schema.yml` for All Models

One `schema.yml` per directory (or per model for complex models):

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: dim_customer
    description: "One row per customer. SCD Type 1 — no history tracking."
    config:
      contract:
        enforced: true
    columns:
      - name: customer_sk
        description: "Surrogate key for the customer dimension."
        data_type: string
        tests:
          - not_null
          - unique
      - name: customer_bk
        description: "Business key from the source CRM system (customer ID)."
        data_type: string
        tests:
          - not_null
      - name: region
        description: "Customer's geographic region."
        data_type: string
        tests:
          - accepted_values:
              values: ['APAC', 'EMEA', 'AMER', 'LATAM']

  - name: fct_orders_placed
    description: "One row per order line item. Grain: one product on one order."
    columns:
      - name: order_sk
        description: "Surrogate key for the order line fact."
        tests: [not_null, unique]
      - name: customer_sk
        description: "FK to dim_customer."
        tests:
          - not_null
          - relationships:
              to: ref('dim_customer')
              field: customer_sk
      - name: revenue_usd
        description: "Gross revenue in USD. Fully additive."
        tests: [not_null]
```

## Step 8: Generate `dbt_project.yml` Additions

```yaml
# Add to dbt_project.yml under 'models:'
models:
  your_project_name:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
      fct_orders_placed:
        +materialized: incremental
```

## Step 9: Output File Summary

Present a complete list of all generated files:

```
models/
├── staging/
│   └── <source>/
│       ├── sources.yml
│       ├── stg_<source>__orders.sql
│       ├── stg_<source>__customers.sql
│       └── stg_<source>__products.sql
├── intermediate/
│   ├── int_orders_with_customer.sql
│   └── int_orders_with_products.sql
└── marts/
    ├── schema.yml
    ├── dim_customer.sql
    ├── dim_product.sql
    └── fct_orders_placed.sql
```

---

## Self-Check Before Finalising

- [ ] Every `{{ source() }}` call references a table declared in `sources.yml`
- [ ] Every `{{ ref() }}` call references a model that exists in the generated file list
- [ ] No hardcoded schema or table names anywhere in model SQL
- [ ] Surrogate keys use `dbt_utils.generate_surrogate_key()` with explicit business key column list
- [ ] Staging models: no business logic, 1:1 with source tables, materialized as view
- [ ] Fact mart models: use `incremental` materialization with `unique_key`
- [ ] Dimension mart models: use `table` materialization, schema contract enforced
- [ ] `schema.yml` includes `not_null` + `unique` tests on all surrogate keys
- [ ] `relationships` tests declared for all FK columns in fact tables
