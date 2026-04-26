# Data Modelling Claude Skills

This repository is a Claude Code skills library for data modelling. It contains 10 Claude Code skills that guide users through complex data modelling tasks — from designing star schemas and Data Vault models to generating dbt projects and ERDs.

## Skills Location

All skills are in `skills/<skill-name>/SKILL.md`. Claude Code discovers them automatically when this repository (or its `skills/` directory) is added to your project.

## How to Use a Skill

Invoke any skill by typing its name as a slash command:

```
/star-schema-generator
/data-model-assessor
/dbt-model-builder
```

You can append a dialect override to any skill:

```
/star-schema-generator use Snowflake SQL dialect
/model-optimizer use PostgreSQL
```

## Default Behaviour

- **Default SQL dialect:** BigQuery (Standard SQL). Override at invocation time.
- **Shared reference files:** Before executing any skill, Claude reads `shared/glossary.md` for canonical terminology. Skills that generate DDL also read `shared/sql-dialect-guide.md`.
- **Clarification over guessing:** When required inputs are missing, Claude will ask clarifying questions rather than assume.
- **Naming conventions:** All generated object names follow `shared/naming-conventions.md`.

## Available Skills

| Skill | Invoke | Purpose |
|---|---|---|
| star-schema-generator | `/star-schema-generator` | Generate Kimball star schema from business requirements |
| snowflake-schema-designer | `/snowflake-schema-designer` | Design normalized snowflake schemas with hierarchical dimensions |
| data-model-assessor | `/data-model-assessor` | Score and review an existing schema against best practices |
| model-optimizer | `/model-optimizer` | Recommend indexes, partitioning, clustering, and denormalization |
| dbt-model-builder | `/dbt-model-builder` | Generate a full dbt project layer from raw source tables |
| scd-designer | `/scd-designer` | Design SCD Type 1/2/3/4/6 with merge SQL and dbt snapshots |
| data-vault-modeller | `/data-vault-modeller` | Convert schemas to Data Vault 2.0 (Hubs, Links, Satellites) |
| erd-generator | `/erd-generator` | Generate Mermaid ERDs from SQL DDL or table descriptions |
| normalization-advisor | `/normalization-advisor` | Analyse normal form compliance and OLTP vs OLAP trade-offs |
| grain-definer | `/grain-definer` | Define fact table grain, detect fanout/chasm traps, validate measures |

## Skill Chaining

Recommended invocation sequences for common use cases:

| Use Case | Sequence |
|---|---|
| New OLAP project | `/grain-definer` → `/star-schema-generator` → `/erd-generator` → `/dbt-model-builder` |
| Assess + fix existing model | `/data-model-assessor` → `/model-optimizer` |
| Add history to a dimension | `/scd-designer` → `/dbt-model-builder` |
| Build Data Vault layer | `/data-vault-modeller` → `/erd-generator` |
| Schema health check | `/normalization-advisor` → `/data-model-assessor` |

## Shared Reference Files

- `shared/glossary.md` — canonical definitions for all data modelling terms used across skills
- `shared/naming-conventions.md` — naming standards for all generated objects
- `shared/sql-dialect-guide.md` — DDL syntax variations across BigQuery, Snowflake, Redshift, PostgreSQL
