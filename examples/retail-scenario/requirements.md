# Example Scenario: Online Retail

Use this scenario as a standard test input for validating the skills in this library.

---

## Business Context

**Company:** GlobalCart — an online and in-store retail business selling consumer electronics, clothing, and home goods.

**Systems:**
- `salesforce_crm` — Customer records (customer_id, name, email, phone, address, segment, region)
- `erp_products` — Product catalogue (product_sku, product_name, description, cost_price, category_id, subcategory_id, brand_id, weight_kg)
- `pos_orders` — Point-of-sale and online orders (order_id, customer_id, store_id, order_date, ship_date, status, total_amount, tax_amount, shipping_amount)
- `pos_order_lines` — Individual line items per order (order_id, line_number, product_sku, quantity, unit_price, discount_amount, promotion_id)
- `erp_stores` — Store locations (store_id, store_name, store_type: 'online'/'physical', city, state, country, region)
- `erp_promotions` — Active promotions (promotion_id, promotion_name, promotion_type, discount_percentage, start_date, end_date)

---

## Business Process to Model

**Process:** Sales — when customers purchase products

**Reporting requirements:**
- Revenue by product category, region, and time period
- Customer segment analysis over time (customers change segment quarterly)
- Promotion effectiveness (revenue uplift from promotions)
- Store performance comparison (online vs physical)
- Product sell-through rates (quantity sold vs stock level)

---

## Proposed Grain

**One row per order line item** — each row represents one product sold on one order.

---

## Key Entities

| Entity | Source Table | Business Key | Changes Over Time? |
|---|---|---|---|
| Customer | `salesforce_crm.customers` | `customer_id` | Yes — address, segment, region change. Need SCD Type 2 |
| Product | `erp_products.products` | `product_sku` | Rarely — name or price corrections only (SCD Type 1) |
| Store | `erp_stores.stores` | `store_id` | Rarely (SCD Type 1) |
| Promotion | `erp_promotions.promotions` | `promotion_id` | No history needed |
| Date | Generated | `full_date` | No |
| Order | `pos_orders.orders` | `order_id` | Degenerate dimension (no descriptive attributes beyond FK to fact) |

---

## Product Hierarchy

Product has a four-level hierarchy for the snowflake schema skill:

```
dim_product
  └── dim_subcategory (subcategory_id, subcategory_name, category_id)
        └── dim_category (category_id, category_name, department_sk)
              └── dim_department (department_id, department_name, region)
```

Approximate cardinality:
- Products: 50,000
- Subcategories: 200
- Categories: 40
- Departments: 8

---

## Key Metrics (Measures)

| Measure | Column | Additivity | Type |
|---|---|---|---|
| Revenue | `revenue_usd` = `unit_price × quantity - discount_amount` | Fully additive | NUMERIC(18,2) |
| Quantity | `quantity_units` | Fully additive | INT |
| Discount amount | `discount_amount_usd` | Fully additive | NUMERIC(18,2) |
| Tax amount | `tax_amount_usd` | Fully additive | NUMERIC(18,2) |
| Discount % | Derived: `discount_amount / (unit_price × quantity)` | Non-additive | Do NOT store as column |

---

## Known Issues to Test

The following issues are intentionally present to validate skill detection capabilities:

1. **Promotion M:M relationship** — A single order line can have multiple promotions applied. This should trigger fanout trap detection in `/grain-definer` and bridge table recommendation in `/star-schema-generator`.

2. **Customer SCD requirement** — Customer segment changes quarterly. This should trigger SCD Type 2 recommendation in `/scd-designer`.

3. **Non-additive measure** — Discount percentage is a ratio and should not be stored. `/grain-definer` and `/star-schema-generator` should flag this.

4. **Role-playing date dimension** — Both `order_date` and `ship_date` exist. `/star-schema-generator` should generate two FK columns referencing `dim_date` with alias views.

5. **Product snowflake hierarchy** — The four-level product hierarchy should be detected by `/snowflake-schema-designer`.

---

## Expected Skill Outputs

Invoke each skill with this scenario to validate it works correctly:

| Skill | Expected Key Output |
|---|---|
| `/grain-definer` | Grain: "one order line item per product on one order"; promotion fanout trap detected |
| `/star-schema-generator` | 5 dimensions (customer, product, store, promotion, date), 1 fact, bridge table for promotions |
| `/snowflake-schema-designer` | 4-level product hierarchy: product → subcategory → category → department |
| `/erd-generator` | Mermaid diagram with 6 entities and all FK relationships |
| `/dbt-model-builder` | 5 staging models, 2 intermediate models, dim_customer + dim_product + fct_orders_placed marts |
| `/scd-designer` | SCD Type 2 for customer (segment/address); dbt snapshot YAML |
| `/data-model-assessor` | Assess the raw source tables; identify missing keys and normalization issues |
| `/normalization-advisor` | Identify 3NF violations in source tables (e.g. region derived from store) |
| `/model-optimizer` | Partition fct_orders_placed by order_date; cluster by customer_sk; materialize daily_sales_by_region |
| `/data-vault-modeller` | hub_customer, hub_product, hub_order, lnk_order_customer, lnk_order_product, sat_customer_crm, sat_product_erp, sat_order_pos |
