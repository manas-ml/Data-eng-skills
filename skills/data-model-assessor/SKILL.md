---
name: data-model-assessor
description: Reviews an existing SQL schema or data model and produces a structured assessment report. Scores the model against Kimball and Inmon best practices, identifies anti-patterns, and provides actionable improvement recommendations. Use when the user provides SQL DDL, an ERD description, or table definitions to be evaluated.
---

# Data Model Assessor

## Step 1: Load Shared Resources

Before proceeding, read:
- `shared/glossary.md`
- `shared/naming-conventions.md`

## Step 2: Parse Input

Accept any of the following input formats:
- SQL `CREATE TABLE` DDL statements
- Prose description of table names, columns, and relationships
- Mermaid `erDiagram` block

From the input, extract:
- List of all table names
- For each table: column names, data types, primary key(s), foreign key references
- Explicit relationship declarations (FK constraints)
- Any indexes, partition keys, or constraints declared

If the input is ambiguous or incomplete, ask the user to clarify before scoring.

## Step 3: Classify the Model Type

Based on the schema structure, classify the model as one of:

| Type | Indicators |
|---|---|
| **Kimball Star Schema** | Tables with `fct_` prefix, wide denormalised dimensions, surrogate keys in fact table |
| **Kimball Snowflake Schema** | As above, but dimensions chain to sub-dimensions via FKs |
| **Inmon / 3NF OLTP** | Fully normalised, many small tables, natural keys as PKs |
| **Data Vault 2.0** | Tables with `hub_`, `lnk_`, `sat_` prefixes; hash keys; `load_date`, `record_source` columns |
| **Flat / Denormalised** | Few tables, repeated values, no FK constraints |
| **Mixed / Unclear** | Elements of multiple styles |

State the classification and the evidence for it. Invite the user to correct the assumption if wrong.

**Important:** Score the model relative to its stated purpose, not against an arbitrary standard. A well-designed 3NF OLTP schema should score highly on its own merits, even if it scores poorly on Kimball-specific criteria.

## Step 4: Score the Model

Score each dimension 0–5 using the rubric below. Apply the rubric honestly — do not inflate scores.

### Scoring Rubric

**1. Grain Clarity (0–5)**
- 5: Grain is inferable from table name + columns; one row = one clearly defined event
- 3: Grain is guessable but not obvious; some ambiguity
- 1: Multiple entities mixed in one table; grain unclear
- 0: No discernible grain; fact and dimension data mixed without pattern

**2. Key Management (0–5)**
- 5: Surrogate PKs on all dimensions; fact table uses surrogate FKs only; natural keys have UNIQUE constraints
- 3: Mostly surrogate keys but some natural keys used as PKs or FKs
- 1: Natural keys used as PKs; compound natural keys used as fact FKs
- 0: No primary keys declared; or identity columns used as business keys

**3. Naming Conventions (0–5)**
- 5: Consistent, descriptive naming throughout; follows a recognisable convention (e.g. `fct_`, `dim_`, `stg_`, snake_case)
- 3: Mostly consistent with minor deviations
- 1: Mixed conventions (camelCase + snake_case, abbreviations mixed with full words)
- 0: No apparent convention; cryptic or confusing names

**4. Normalization Appropriateness (0–5)**
- 5: Normalization level is correct for the stated purpose (3NF for OLTP; denormalised dimensions for OLAP)
- 3: Mostly appropriate with some over- or under-normalization
- 1: Dimension tables are over-normalized for OLAP (queries require many joins); or OLTP tables have repeating groups
- 0: Flat table design used for high-write transactional workloads; or fully normalised OLAP with no dimensional layer

**5. Referential Integrity (0–5)**
- 5: All FK relationships declared with constraints; no orphan FK columns
- 3: Most relationships declared; a few FK columns without constraints
- 1: FK relationships implied by column names but not declared
- 0: No FK constraints; relationships inferred only from documentation (or none)

**6. Measure Quality (0–5)**
- 5: Measures are appropriately typed (NUMERIC not FLOAT for money); additivity type is documented; no ratios stored
- 3: Measures are reasonably typed; additivity not documented; some FLOAT used for currency
- 1: Ratios/percentages stored as measures; FLOAT used for currency; mixed measure types
- 0: Measures not separable from descriptive attributes; or no numeric columns

**7. SCD / History Tracking (0–5)**
- 5: `effective_date`, `expiry_date`, `is_current` present on dimensions that change; or documented as Type 1 intentionally
- 3: Some history columns present but incomplete (e.g. `updated_at` only)
- 1: No history mechanism; dimension changes would silently overwrite historical records
- 0: History required but no mechanism exists and no acknowledgement of the gap

**8. Documentation (0–5)**
- 5: Column descriptions present (in schema.yml, comments, or data dictionary); table purpose documented
- 3: Some tables documented; column-level descriptions sparse
- 1: Table names and column names only; no descriptions
- 0: No documentation; column names are cryptic

### Score Summary Table

| Dimension | Score (0–5) | Evidence / Notes |
|---|---|---|
| Grain Clarity | | |
| Key Management | | |
| Naming Conventions | | |
| Normalization Appropriateness | | |
| Referential Integrity | | |
| Measure Quality | | |
| SCD / History Tracking | | |
| Documentation | | |
| **Total** | **/40** | |

**Letter grade:**
- 36–40: A (Excellent)
- 30–35: B (Good)
- 22–29: C (Acceptable)
- 14–21: D (Needs improvement)
- 0–13:  F (Major issues)

## Step 5: Anti-Pattern Checklist

Check for each of the following and flag any that are present:

| Anti-Pattern | Present? | Tables/Columns Affected |
|---|---|---|
| Many-to-many relationship without a bridge table | | |
| NULL foreign keys in fact table without documented reason | | |
| Compound natural key used as fact table FK | | |
| Degenerate dimensions stored in a separate table unnecessarily | | |
| Role-playing dimensions not aliased (single `date_sk` for multiple date roles) | | |
| Fact-on-fact joins (one fact table FK-ing another fact table) | | |
| Surrogate key not used as primary key of dimension | | |
| FLOAT data type used for monetary amounts | | |
| Repeating column groups (e.g. `address_1`, `address_2`, `address_3`) | | |
| Catch-all "description" or "misc" columns of type STRING/TEXT | | |
| Timestamp columns used as primary keys | | |
| Measure columns with no data type precision (e.g. unqualified `DECIMAL`) | | |

## Step 6: Recommendations

For each issue identified in Steps 4 and 5, provide a specific recommendation:

| # | Issue | Severity | Table / Column | Recommended Fix |
|---|---|---|---|---|
| 1 | | HIGH / MEDIUM / LOW | | |

Sort recommendations by severity (HIGH first).

## Wrap-Up Output Format

---

## Data Model Assessment Report

### Model Classification
**Type:** `<Kimball Star / Snowflake / 3NF OLTP / Data Vault / Flat / Mixed>`
**Evidence:** ...
**Assumed purpose:** `<OLAP reporting / OLTP transactions / Mixed>` — *please correct if this assumption is wrong*

### Score Summary
| Dimension | Score | Notes |
|---|---|---|
| Grain Clarity | x/5 | |
| Key Management | x/5 | |
| Naming Conventions | x/5 | |
| Normalization Appropriateness | x/5 | |
| Referential Integrity | x/5 | |
| Measure Quality | x/5 | |
| SCD / History Tracking | x/5 | |
| Documentation | x/5 | |
| **Total** | **x/40** | |

**Grade: `<A/B/C/D/F>`**

### Anti-Pattern Findings
| Anti-Pattern | Tables / Columns |
|---|---|

### Recommendations
| # | Issue | Severity | Location | Fix |
|---|---|---|---|---|

### Executive Summary
[One paragraph summarising the overall quality of the model, the top 2–3 most important issues to address, and the expected improvement from doing so.]

---

## Self-Check Before Finalising

- [ ] Model type is classified and assumption is stated for user to confirm
- [ ] All 8 scoring dimensions are assessed with evidence, not just a number
- [ ] Anti-pattern checklist is fully completed (mark each as Present / Not Present)
- [ ] Every HIGH severity recommendation references a specific table and column
- [ ] Executive summary is written for a non-technical stakeholder
- [ ] Score is honest — do not inflate scores to be encouraging
