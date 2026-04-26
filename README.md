# Data Engineering Skills — Data Modelling Library

A Claude Code skills library with 10 production-grade data modelling skills. Each skill is a structured prompt that guides Claude through complex data modelling workflows — generating schemas, assessing models, building dbt projects, designing SCDs, and more.

## Quick Start

### 1. Install

Copy the `skills/` directory into your project's `.claude/` folder, or set the `CLAUDE_SKILLS_PATH` environment variable to point to this repository.

```bash
cp -r skills/ /your-project/.claude/skills/
```

Or reference the whole repo:

```bash
export CLAUDE_SKILLS_PATH=/path/to/Data-eng-skills/skills
```

### 2. Invoke a Skill

In Claude Code, type the skill name as a slash command:

```
/star-schema-generator
```

Claude will walk you through the required inputs and produce the output.

---

## The 10 Skills

### 1. `/star-schema-generator`
Generates a complete Kimball-style star schema from business requirements.

**Produces:** Grain statement, dimension table designs, fact table design, SQL DDL, dbt-style schema.yml documentation.

**Example input:** "I need a sales schema for our e-commerce business. Entities: Customer, Product, Order, Store, Date. Grain: one row per order line."

---

### 2. `/snowflake-schema-designer`
Designs normalized snowflake schemas for dimensions with multi-level hierarchies.

**Produces:** Sub-dimension chain DDL, ERD outline, trade-off analysis vs star schema, flattened materialized view.

**Example input:** "My Product dimension has: Product → Subcategory → Category → Department. Normalize it."

---

### 3. `/data-model-assessor`
Reviews an existing schema against Kimball and Inmon best practices. Produces a scored assessment report.

**Produces:** Score table (0–40, A–F grade), anti-pattern checklist, recommendations table, executive summary.

**Example input:** Paste your `CREATE TABLE` DDL or describe your table structure.

---

### 4. `/model-optimizer`
Analyzes a schema and query patterns to recommend performance optimizations.

**Produces:** Index DDL, partitioning strategy, clustering/sort key recommendations, materialization plan, denormalization opportunities — all sorted by impact vs effort.

**Example input:** "Here's my schema and 3 slow queries. Platform: BigQuery. Row counts: orders = 500M, customers = 10M."

---

### 5. `/dbt-model-builder`
Converts raw source tables into a complete dbt project layer.

**Produces:** `sources.yml`, staging models, intermediate models, mart models, `schema.yml` with tests, `dbt_project.yml` additions.

**Example input:** "Here are my raw tables from Salesforce. I need a customer_360 mart."

---

### 6. `/scd-designer`
Designs Slowly Changing Dimension implementations with full DDL and CDC merge logic.

**Produces:** SCD type recommendation, DDL for chosen type(s), MERGE/upsert SQL, dbt Snapshot YAML, validation queries.

**Example input:** "My Customer dimension changes address and segment. I need full history for reporting."

---

### 7. `/data-vault-modeller`
Converts a relational or dimensional model into Data Vault 2.0 patterns.

**Produces:** Hub DDL, Link DDL, Satellite DDL, hash key SQL, load INSERT patterns, optional PIT table.

**Example input:** "I have Orders, Customers, and Products from 3 source systems. Business keys: customer_id, order_id, product_sku."

---

### 8. `/erd-generator`
Generates Mermaid Entity-Relationship Diagrams from SQL DDL or prose descriptions.

**Produces:** Fenced Mermaid `erDiagram` block, relationship verification table, rendering instructions.

**Example input:** Paste your SQL DDL or describe your tables and relationships.

---

### 9. `/normalization-advisor`
Analyses a schema for 1NF–BCNF compliance and provides normalization or denormalization recommendations.

**Produces:** Normal form test results, recommendations with refactoring DDL, OLTP vs OLAP trade-off analysis, migration risk assessment.

**Example input:** "Here's my flat customer table with repeated order columns. It's used for OLTP."

---

### 10. `/grain-definer`
Helps define and validate fact table grain; detects fanout and chasm traps; validates measure additivity.

**Produces:** Grain statement, conformed dimension classification, fanout/chasm diagnostic SQL, measure additivity classification, validation SQL.

**Example input:** "My sales fact table keeps giving inflated revenue totals when I join the promotion dimension."

---

## Skill Chaining

Combine skills for end-to-end workflows:

| Use Case | Sequence |
|---|---|
| New OLAP project from scratch | `/grain-definer` → `/star-schema-generator` → `/erd-generator` → `/dbt-model-builder` |
| Assess and fix an existing model | `/data-model-assessor` → `/model-optimizer` |
| Add history tracking to a dimension | `/scd-designer` → `/dbt-model-builder` |
| Build a Data Vault layer | `/data-vault-modeller` → `/erd-generator` |
| Schema health check | `/normalization-advisor` → `/data-model-assessor` |

---

## SQL Dialect Support

All DDL-generating skills support these platforms:

- **BigQuery** (default)
- **Snowflake**
- **Amazon Redshift**
- **PostgreSQL**

Override the dialect at invocation: `/star-schema-generator use Snowflake SQL`

---

## Repository Structure

```
├── CLAUDE.md                        # Claude Code project configuration
├── README.md                        # This file
├── shared/
│   ├── glossary.md                  # Canonical data modelling terminology
│   ├── naming-conventions.md        # Naming standards for all generated objects
│   └── sql-dialect-guide.md         # Platform-specific DDL syntax reference
├── skills/
│   ├── star-schema-generator/SKILL.md
│   ├── snowflake-schema-designer/SKILL.md
│   ├── data-model-assessor/SKILL.md
│   ├── model-optimizer/SKILL.md
│   ├── dbt-model-builder/SKILL.md
│   ├── scd-designer/SKILL.md
│   ├── data-vault-modeller/SKILL.md
│   ├── erd-generator/SKILL.md
│   ├── normalization-advisor/SKILL.md
│   └── grain-definer/SKILL.md
└── examples/
    ├── retail-scenario/requirements.md
    └── healthcare-scenario/requirements.md
```

---

## Contributing a New Skill

1. Create `skills/<skill-name>/SKILL.md`
2. Add YAML frontmatter with `name:` and `description:`
3. Open with "Step 1: Load shared resources" referencing the relevant shared files
4. Define a clear, numbered workflow with imperative steps
5. End with a "Wrap-up output format" and "Self-check before finalizing" section
6. Add an example invocation to this README
