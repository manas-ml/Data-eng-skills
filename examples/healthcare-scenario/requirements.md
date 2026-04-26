# Example Scenario: Healthcare Claims

Use this scenario to test skills with a more complex, regulated domain.

---

## Business Context

**Organisation:** MedInsure — a health insurance company processing medical claims.

**Systems:**
- `claims_system` — Claims (claim_id, patient_id, provider_id, plan_id, submission_date, adjudication_date, status, total_billed, total_allowed, total_paid)
- `claims_lines` — Claim line items (claim_id, line_number, procedure_code, diagnosis_code, service_date, service_from, service_to, units, billed_amount, allowed_amount, paid_amount)
- `member_system` — Members/patients (member_id, member_name, dob, gender, plan_id, enrollment_date, termination_date, address, region)
- `provider_registry` — Providers (provider_id, provider_name, provider_type, specialty, npi, network_status, address, state)
- `plan_master` — Insurance plans (plan_id, plan_name, plan_type, metal_tier, deductible_amount, out_of_pocket_max)
- `reference_icd10` — ICD-10 diagnosis codes (icd10_code, description, chapter, major_category, sub_category)
- `reference_cpt` — CPT procedure codes (cpt_code, description, category, sub_category)

---

## Business Process to Model

**Process:** Claims adjudication — the lifecycle of a medical claim from submission to payment

**Reporting requirements:**
- Claim payment analysis by diagnosis category, provider type, and plan
- Member utilisation trends over time
- Provider network performance (in-network vs out-of-network cost)
- Denial rate analysis by denial reason, procedure code, and provider
- Member risk profiling (cumulative claims by member)

---

## Proposed Grain Options

This scenario is intentionally ambiguous to test `/grain-definer`:

**Option A — Claim level:** One row per claim
- Pro: Simple; matches claim header reporting
- Con: Loses detail about individual procedures

**Option B — Claim line level:** One row per claim line (one procedure on one claim)
- Pro: Maximum granularity; procedure-level analysis possible
- Con: Aggregating to claim level requires SUM or COUNT DISTINCT

**Expected outcome from `/grain-definer`:** Recommend Option B (claim line) with a note that claim-level aggregations need `COUNT(DISTINCT claim_id)` not `COUNT(*)`.

---

## Key Entities

| Entity | Source Table | Business Key | Changes Over Time? |
|---|---|---|---|
| Member | `member_system.members` | `member_id` | Yes — plan_id, address change; enrollment/termination dates |
| Provider | `provider_registry.providers` | `provider_id` (NPI) | Yes — network_status changes (in/out of network) |
| Plan | `plan_master.plans` | `plan_id` | Rarely — deductible/OOP max changes annually |
| Diagnosis | `reference_icd10` | `icd10_code` | No (reference table) |
| Procedure | `reference_cpt` | `cpt_code` | No (reference table) |
| Date | Generated | `full_date` | No |
| Claim | `claims_system.claims` | `claim_id` | Degenerate dimension |

---

## Key Metrics

| Measure | Column | Additivity |
|---|---|---|
| Billed amount | `billed_amount_usd` | Fully additive |
| Allowed amount | `allowed_amount_usd` | Fully additive |
| Paid amount | `paid_amount_usd` | Fully additive |
| Denied amount | `billed_amount - paid_amount` | Derived — store components |
| Units of service | `units` | Fully additive |
| Approval rate | `paid_amount / billed_amount` | Non-additive — derive in BI layer |
| Cumulative member costs | Running SUM of `paid_amount_usd` per member | Semi-additive — cannot SUM across members and time simultaneously |

---

## Data Vault Use Case

This healthcare scenario is ideal for the Data Vault pattern because:
- Claims come from multiple source systems (hospitalisation, pharmacy, dental — separate feeds)
- Regulatory requirements mandate full audit history
- Members can appear in multiple source systems with different IDs

**Business keys for Data Vault:**
- Member: `member_id` (internal) — note: same member may have `member_id_dental`, `member_id_pharmacy` in other systems
- Provider: `npi` (National Provider Identifier — universal across systems)
- Claim: `claim_id` (source-system-specific — must carry `record_source` to distinguish)
- Plan: `plan_id`

---

## Known Issues to Test

1. **Semi-additive cumulative costs** — `/grain-definer` should flag `cumulative_paid_amount_usd` as semi-additive and prescribe correct aggregation.

2. **Multiple date dimensions** — Claims have `submission_date`, `adjudication_date`, `service_date` (from claim line), `service_from`, `service_to`. `/star-schema-generator` should generate role-playing date FKs for each.

3. **ICD-10 and CPT are reference dimensions** — Low-cardinality by category, high-cardinality by individual code. `/star-schema-designer` should consider whether to snowflake ICD-10 into chapter → major_category → sub_category → code.

4. **Multi-source member identity** — `/data-vault-modeller` should model this as separate Hubs per source system with a Link connecting the business identities.

5. **Plan changes annually** — `/scd-designer` should recommend SCD Type 2 for plan deductible/OOP max changes, but SCD Type 1 for plan name corrections.

---

## Expected Skill Outputs

| Skill | Expected Key Output |
|---|---|
| `/grain-definer` | Recommend claim-line grain; flag semi-additive cumulative costs; flag approval_rate as non-additive |
| `/star-schema-generator` | 6 dimensions (member, provider, plan, diagnosis, procedure, date), 1 fact, 5 role-playing date FKs |
| `/data-vault-modeller` | hub_member, hub_provider, hub_claim, hub_plan, lnk_claim_member, lnk_claim_provider, lnk_claim_procedure, lnk_claim_diagnosis + satellites per source |
| `/scd-designer` | SCD Type 2 for member (plan_id, address) and provider (network_status); SCD Type 1 for plan name |
| `/dbt-model-builder` | 7 staging models (claims, lines, member, provider, plan, icd10_ref, cpt_ref), 3 intermediate, mart fact + 5 dimensions |
