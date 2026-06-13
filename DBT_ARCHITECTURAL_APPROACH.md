# dbt Architectural Approach: Analytical Layers from Scattered Raw Operational Data

**Concept Document — Lightspeed / `odoo_earth` Proof of Concept**
*Audience: Technical leadership, data team leads, and decision-makers evaluating analytics infrastructure*

---

## Overview

This document uses a real, completed proof-of-concept — built against a single company database — to make the case for a layered dbt analytics architecture applied across all operational data sources. The sample is not the full picture. It is a demonstration of a repeatable pattern that solves four hard problems simultaneously: inconsistent KPI numbers across dashboards, undocumented and scattered query logic, hidden data quality traps, and the absence of a trusted layer for reporting.

Every number in this document comes from a direct live probe against the `odoo_earth` PostgreSQL database. No estimates.

---

## Part 1 — The Sample

### What Was Given

Access to a single live PostgreSQL database: `odoo_earth`. This is the company's Odoo ERP instance. No data dictionary. No entity-relationship diagram. No documentation. Just a live operational system.

**What the database actually contains:**

| Metric | Value |
|---|---|
| Total tables | 1,027 |
| Empty tables (0 rows) | 662 (64%) |
| Total database size | 1,448 MB |
| Largest single namespace | `mail_*` — 904 MB, pure system noise |
| Tables relevant to field operations | ~26 (identified after systematic probing) |

Of 1,027 tables, **64% have zero rows**. They exist because Odoo installs schema for every module it ships, whether or not that module was ever activated. A dashboard built against this database without systematic curation would reference columns that sound meaningful but are permanently empty.

### What Was Built

Over four weeks, the following was produced through a systematic probe-first, model-second workflow:

| Layer | Count | Schema |
|---|---|---|
| Staging models (raw → cleaned, renamed, typed) | 26 | `dbt_dev_staging` |
| Intermediate models (business logic, enrichment) | 3 | `dbt_dev_intermediate` |
| Mart fact table (dashboard-ready) | 1 | `dbt_dev_marts` |
| Declarative YAML tests | 171 | Distributed across all layers |
| Build test results (latest run) | **32 PASS · 0 WARN · 0 ERROR** | — |
| Grafana dashboard panels built on the mart | 42 | All reading from a single fact table |
| Source table documentation | 100% model coverage, ~700 column descriptions | YAML |

The single output fact table — `fct_field_service_work_orders` — contains **10,554 rows** (one per field work order), **77 columns**, and was validated grain-clean: `COUNT(*) = COUNT(DISTINCT maintenance_request_id) = 10,554`. Zero duplicate rows. Zero fanout from joins.

### What the Database Is

Before any modeling, systematic Python probes (`information_schema.tables`, `information_schema.columns`, Odoo's internal `ir_model_fields` registry) revealed the most important architectural fact: **`odoo_earth` is not a standalone ERP. It is operational middleware.**

The actual systems of record are three external platforms:

| External System | Role | What It Owns |
|---|---|---|
| **Creatio CRM** | Customer-facing contact centre | All subscriber accounts, CC ticket cases, installation orders |
| **FMS** (Fiber Management System, company-built) | Network provisioning and billing | Contracts, zone/FAT topology, port assignments, subscriber IDs |
| **HRIS** | Human resources | Full employee database, org hierarchy, payroll |

Odoo's role: dispatch field teams, log work orders, track technician actions, and sync status to/from Creatio and FMS. It is the field operations coordination layer, not the subscriber database and not the billing system.

**Why this matters:** Any traditional dashboard built against `odoo_earth` alone is analyzing a middleware layer. The subscriber universe is in FMS. The CC case classification is in Creatio. Costs are in FMS billing. A raw-DB query that ignores this architecture will produce answers that look plausible but are structurally incomplete.

---

## Part 2 — Why Traditional Direct-DB Dashboard Approaches Fail

The following failures were observed directly in the existing dashboard SQL and probe results. These are not theoretical risks. They happened.

### 2.1 Same Database, Different KPI Numbers

**The subscriber count problem**

A direct `SELECT COUNT(*) FROM res_partner` returns **27,115**. This looks like the subscriber count. It is not. A probe using Odoo's own flags breaks the population down:

| Population | Count |
|---|---|
| Real subscribers (`sap_user = TRUE`) | 19,194 |
| Unclassified contacts (no flags) | 5,562 |
| Zone contractors | 184 |
| Internal staff | 94 |
| Vendors | 63 |
| **Total** | **27,115** |

The naive count is **41% higher** than the subscriber count. Every dashboard metric expressed as "per subscriber" or "% of customers" will be wrong unless this filter is centralized.

**The monthly trend artifact**

The existing Grafana SQL used `COUNT(mr.partner_id)` as a proxy for "customers served this month." Because `partner_id` on `maintenance_request` is only **22.3% populated overall** — and that population rate varies massively by period as data-entry habits evolved — this query produces a trend that looks like business growth but is actually a data-entry artifact:

| Month | Records | `partner_id` filled | Fill rate |
|---|---|---|---|
| Feb 2026 | 521 | 14 | **2.7%** |
| Apr 2026 | 3,536 | 2,602 | **73.6%** |

A board presentation using this series would show a 27× spike in "customers served" from February to April. That spike is not real. The subscriber identity field for this workflow is not `partner_id` — it is `contract_sap_id` and `username`. Without documented field semantics, this gap is invisible.

**The `sap_id` cross-system trap**

The column `sap_id` appears on four different tables. Every other engineer on this project documented it as "SAP ERP system reference ID." Direct probes showed:

| Table | `sap_id` Format | Actual Source | What It Is |
|---|---|---|---|
| `helpdesk_ticket` | UUID (36 chars) | **Creatio CRM** | CC ticket/case record ID |
| `maintenance_request` | UUID (36 chars) | **Creatio CRM** | Installation order ID (only on installs) |
| `res_partner` | 7-digit numeric | **FMS** | FMS subscriber customer ID |
| `res_zone` | Alphanumeric (e.g. `FBG0462-3`) | **FMS** | FMS zone topology code |

A join between `helpdesk_ticket.sap_id` and `res_partner.sap_id` would join a Creatio UUID to an FMS numeric ID. It returns zero rows. It does not error — it silently produces an empty result, which looks like "no data" rather than "wrong join."

The dbt staging models renamed every occurrence to reflect the actual source system:
- `helpdesk_ticket.sap_id` → `ticket_creatio_uuid`
- `res_zone.sap_id` → `sap_zone_id`
- `res_partner.sap_id` → `fms_customer_id`

**Helpdesk subscriber identity: 1.6% join success**

The existing SQL joined `helpdesk_ticket` to `res_partner` on `partner_id` to identify which subscriber raised each ticket. That join succeeds for **105 of 6,725 tickets (1.6%)**. The CC team identifies subscribers via `contract_sap_id` (66.8% populated) and `username` (FMS port string), not via Odoo's `partner_id` link — which was never activated in the CC workflow. A dashboard built on this join shows the correct subscriber name for 1 in 60 tickets.

---

### 2.2 Silent Data Traps

These are columns or modules that exist in the database, return no errors when queried, and produce values — but those values are wrong or meaningless.

**The `collection` boolean — always FALSE**

`maintenance_request.collection` is a boolean column intended to record whether cash was collected on a field visit. Direct probe:

```sql
SELECT COUNT(*), SUM(CASE WHEN collection THEN 1 ELSE 0 END)
FROM maintenance_request;
→ 10,554 total | 0 with collection = TRUE
```

Every single row is `FALSE`. The column was never activated in this Odoo instance. The correct signal is `collected_amount > 0`, which identifies **2,551 work orders (24.2%)** with cash collected. A KPI dashboard using the `collection` column would show **zero cash collection across all field visits** — a materially incorrect result. The dbt model derives `is_cash_collected` from `collected_amount` and excludes the boolean entirely.

**SLA compliance — module installed, never configured**

The SLA module tables exist: `helpdesk_sla` and `helpdesk_sla_status`. A probe:

```sql
SELECT COUNT(*) FROM helpdesk_sla;        → 0 rows
SELECT COUNT(*) FROM helpdesk_sla_status; → 0 rows
```

Because the module is installed but unconfigured, Odoo's default behavior sets `is_sla_reached = TRUE` on every single ticket. All 6,725 tickets show `is_sla_reached = TRUE`. An executive dashboard built on this column would report **100% SLA compliance** — not because the company performs well, but because the measurement system was never turned on.

**Placeholder geographic zones**

12 zones have codes ending in `8888` or `8822` — internal placeholders for areas not yet commissioned in the network topology. These zones contain **1,775 work orders (16.8% of all work orders)**, including **1,496 completed installations**. They are real work performed by real technicians, but associating them with a genuine geographic zone for network analysis is incorrect. Without a flag, zone-level KPIs mix commissioned and uncommissioned areas silently. The dbt mart exposes `is_placeholder_zone = TRUE` on all 1,775.

**The description field — structured data treated as free text**

`maintenance_request.description` was initially excluded from all analytics with the documented reason: "free-text notes, too large for standard analysis." This is a reasonable default assumption. A targeted probe on 5,082 maintenance work orders revealed something entirely different: the CC mobile app fills the field with a structured template at job dispatch:

```
Customer Information:
 Name: محمد محسن جدوع
 Phone Number: 07778068671
 Zone: FBG2032-5
 FAT: FAT10
 Location: 44.29361340, 33.35282810
 Description: تم التجيك البور طبيعي
```

The FAT (network node) field-level FK `fat_id` on `maintenance_request` is populated for only **24.7% of work orders**. After parsing the description template in a dedicated intermediate model (`int_field_service_description_parse`):

| Source | FAT / Zone coverage |
|---|---|
| `fat_id` FK alone | **24.7%** |
| Description template parse | **96.7%** |
| Gap recovered from description | **~72 percentage points** |

Treating `description` as unstructured free text permanently loses geographic attribution for three-quarters of all work orders.

**Employee HRIS codes: 78.5% are a Python import artifact**

`hr_employee.emp_id` should contain the employee's HRIS identifier for cross-system joins to the HR platform. Direct probe: **2,523 of 3,215 employees (78.5%)** have `emp_id` values like `False1`, `False2`, `False3` — a Python `False` value that was imported as a string instead of NULL. Only **692 employees (21.5%)** have genuine HRIS codes. Any cross-system HR analysis built on this column covers fewer than one in five employees.

---

### 2.3 Scattered and Undocumented Business Logic

The existing dashboard SQL (`ftth_light_speed.sql`) was reviewed query by query. Key structural problems:

**The same filter written four different times**

The stage completion filter (identifying "done" work orders) was hardcoded differently in four separate query blocks. In one block: `stage_id = 9`. In another: `stage_type = 'done'`. In a third: joining to `maintenance_stage` and filtering on the stage name. In a fourth: the filter was missing entirely, causing the query to include canceled and failed work orders in the "completed" count.

Each Grafana panel that uses a different version of this filter will return a different number for the same KPI.

**A syntax-broken query that silently never ran**

```sql
from res_partner as rp
left join maintenance_request as mr on mr.partner_id = rp.id
left join res_zone as rz on rp.zone_id = rz.id
--where (rp.active is not null and rp.active = true)
and mr.task_type = 'maintenance'   -- syntax error: AND without WHERE
```

The `WHERE` clause was commented out and the `AND` was left in — making this query a syntax error. It was never executed. The zone distribution output expected from this query was missing entirely.

**Regex that fails on Arabic input and bleeds into adjacent fields**

```sql
SUBSTRING(description FROM 'Name:\s*([A-Za-z0-9]+)')
```

The character class `[A-Za-z0-9]` matches only Latin letters and digits. Arabic names are entirely excluded — the result is NULL for all subscribers with Arabic names. For records with single-line HTML (no newlines between template fields), the regex bleeds forward and captures everything after "Name:" including "Phone Number: Zone: FBG..." as the customer name.

The dbt intermediate model uses format-anchored regexes for machine-readable fields (zone codes, FAT labels, GPS coordinates) and `SPLIT_PART` between adjacent label boundaries for human-entered fields, handling both multi-line and single-line HTML formats.

**No grain definition, no tests, no NULL-rate documentation**

The existing SQL contains no test layer. There is no check that the main query returns one row per work order (the grain). There is no check that the status flags are mutually exclusive. There is no documentation of which columns are expected to be sparse. When a result looks wrong, there is no baseline to compare against.

---

## Part 3 — The dbt Answer: Four Pain Points, Four Solutions

### 3.1 One KPI = One Definition, Everywhere

In the dbt architecture, every computed business concept is defined exactly once — in the appropriate layer of the transformation chain — and every dashboard reads from the same materialized output.

**Before:** "Completed work orders" is calculated differently in each dashboard panel depending on which engineer wrote the query and which stage column they chose.

**After:** The staging layer maps raw stage names to a controlled vocabulary. The spine intermediate model derives `is_done`, `is_failed`, and `is_canceled` as mutually exclusive boolean flags using `stage.is_completion_stage`. The fact table inherits these flags. All 42 Grafana panels that need a "done" filter write `WHERE stage_type = 'done'` — they are all reading the same column, derived from the same logic.

**Examples of centralized definitions in the fact table:**

| Concept | Old approach | dbt definition |
|---|---|---|
| Job completed | `stage_id = 9` or `stage_type = 'done'` or stage name join | `is_done = TRUE` (from `stage.is_completion_stage`) |
| Cash collected | `collection = TRUE` (always FALSE) | `is_cash_collected` → `collected_amount_iqd > 0` |
| Placeholder zone | No definition existed | `is_placeholder_zone` → `sap_zone_id LIKE '%8888' OR '%8822'` |
| Real subscriber | `COUNT(*) FROM res_partner` (inflated 41%) | `sap_user = TRUE` filter defined in staging |
| Network location | `fat_id` FK only (24.7%) | Description parse recovers to 96.7% |

Changing any of these definitions — for example, if the business redefines "completed" to exclude same-day cancellations — requires editing one model in one place. Every downstream panel updates on the next `dbt run`.

### 3.2 Documented and Centralized Queries

Every model in the dbt project is:
- **Version-controlled** in git — every change to business logic is tracked with author, date, and commit message
- **Documented in YAML** — 100% of models have a top-level description; approximately 700 column-level descriptions are written across the project
- **Lineage-visible** — dbt generates an interactive DAG showing exactly how every column in every dashboard panel traces back to its raw source table

When a new analyst joins the team, they do not read through dashboards and try to reverse-engineer what `LEFT(sap_zone_id, 3)` is doing. They read the model documentation, which states: "Province code derived from the first 3 characters of the FMS zone code — covers 99.99% of all work orders and helpdesk tickets."

When a stakeholder asks "why is this number different from last quarter's report?" — the answer is findable in git history, not in the memory of whoever wrote the original Grafana query.

### 3.3 Business Logic Fixed Once, Adjustable Forever

**The problem it solves:** In the existing approach, the rule "a work order is 'done' if stage_id = 9" is written in 4+ separate dashboard queries. If the staging system is reorganized and the "Done" stage gets a new ID, every query must be found and updated — and there is no way to know how many queries exist or whether all of them were caught.

**How dbt solves this:** The mapping from raw stage IDs to the `stage_type` vocabulary lives in `stg_odoo__maintenance_stage.sql`. One file. If the stage structure changes, one model is updated. All downstream models — spine, action summary, fact table, all 42 Grafana panels — automatically reflect the change when the project is rebuilt.

The same principle applies to:
- New failure reason codes added by admins (the lookup table grew from 7 to 20 entries; the dbt `relationships` test accommodates growth automatically, unlike a hardcoded `accepted_values` list)
- Changes to placeholder zone conventions (a new code pattern can be added to `is_placeholder_zone` in one line)
- Business rule updates (e.g., reclassifying what counts as a "fast dispatch" changes one threshold in one intermediate model)

### 3.4 A Trusted Layer for Analysis

Trust in a data layer is not claimed — it is proven through systematic checks.

**Grain validation:** `COUNT(*) = COUNT(DISTINCT maintenance_request_id) = 10,554`. There are exactly as many fact table rows as there are distinct work orders. No row was duplicated by a join. No work order was silently dropped.

**Test suite:** 171 declarative tests distributed across all model layers. Categories include:
- `unique` and `not_null` on primary keys (every model)
- `accepted_values` on controlled vocabularies (`task_type`, `stage_type`)
- `relationships` on foreign keys (verifying referential integrity between staged models)
- Severity-aware: sparse FKs known to be partially populated (e.g., `fat_id` at 24.7%) are tested with `warn` severity — the gap is documented as known and quantified, not silently accepted

**Build result:** 32 PASS · 0 WARN · 0 ERROR on the latest full project build.

**Probe-backed documentation:** For every data quality caveat in the fact table, there is a corresponding SQL probe result in the project documentation. The 1.6% `partner_id` figure is not an estimate — it is `SELECT COUNT(*) WHERE partner_id IS NOT NULL` run against the live database. When a dashboard consumer asks "why is technician name missing for some installation orders?" the answer is "45.5% coverage — documented at `int_field_service_work_order_spine_findings.md`," not "we're not sure."

**Infrastructure validation caught a silent build bug:** During development, `dbt-fusion` (an experimental build engine) compiled four staging views with an injected `WHERE false LIMIT 0` clause. The views existed, compiled without errors, and returned zero rows on every query — making it appear that all technician name joins had failed. The systematic validation process (grain check, per-column coverage counts) surfaced this immediately. A dashboard-on-raw-SQL approach has no equivalent safety net; such a bug would appear as "technician data missing" with no clear cause.

---

## Part 4 — Additional Benefits When Rolled Out Per Source

The pattern demonstrated on `odoo_earth` applies identically to every other company data source. When deployed at scale across Creatio CRM, FMS, HRIS, and future systems:

### Automated Data Quality Monitoring

dbt tests run on every build. Failures are alerts. If FMS stops syncing subscriber records to Odoo and the `res_partner` row count drops unexpectedly, a `not_null` test on a required dimension column will fail and notify the team — before a dashboard is presented to the board with the wrong numbers.

### PII Management and Access Control

The dbt YAML layer is where PII columns are documented (`desc_subscriber_name`, `desc_subscriber_phone`, `isp_port_string` are already flagged in the current project). Access control policies can be applied at the mart layer, giving analysts access to aggregated facts without exposing raw contact data. This is impossible in a model where dashboards query raw operational tables directly.

### Cross-Source Conformed Dimensions

Each operational system uses different identifiers for the same real-world entity. FMS uses a 7-digit numeric subscriber ID. Creatio uses a UUID. Odoo uses `res_partner.id`. A dbt conformed dimension table (`dim_subscriber`) can map all three systems' keys to a single business ID — unlocking cross-source metrics like "time from CC ticket creation to field resolution to billing update" that are impossible when each system is queried in isolation.

### Environment Separation (Dev / Staging / Prod)

The current project already runs in `dbt_dev_*` schemas. Promoting to production (`dbt_prod_*`) is a configuration change. This means analysts can develop and test new metrics in a parallel environment without touching the dashboards the board reads. No more "don't run that query on this database, the dashboards will break."

### Incremental Refresh at Scale

Operational databases grow. `maintenance_action_log` already has 39,343 rows from under a year of data. `hr_attendance_zk_temp` has 141,601 rows. FMS may contribute millions of subscriber event rows. dbt's incremental materialization strategy processes only new or changed rows per run, keeping refresh times fast as data volumes grow.

### Onboarding Speed

A new data engineer or analyst who joins the team today can run `dbt docs generate` and browse an interactive catalog of every table, every column, every test, and every lineage relationship. They understand the data model in hours, not weeks. Without the dbt layer, understanding `odoo_earth` required four weeks of systematic probing to map out 26 analytically relevant tables from 1,027 total.

### Audit Trail and Rollback

Every change to business logic goes through a git commit. If a metric definition changes and a stakeholder notices that a KPI shifted, the git log shows exactly which model was changed, when, by whom, and why. If the change was wrong, it can be reverted. In the raw-SQL-on-Grafana model, dashboard definitions are often not version-controlled at all.

### Semantic Layer and AI Readiness

A dbt mart with documented columns and consistent naming conventions is directly consumable by semantic layer tools (Metabase, Looker, Cube.js) and LLM-powered analytics assistants. The consistent column naming (`is_done`, `cycle_time_days`, `team_name`) makes natural-language queries ("show me teams with the lowest completion rate in Baghdad") directly resolvable. Raw operational table queries require an LLM to understand Odoo's internal schema — a far more brittle proposition.

---

## Part 5 — The Rollout Pattern

The same five-step process that produced `fct_field_service_work_orders` in four weeks applies to every remaining domain:

```
Step 1: PROBE        Run systematic Python/SQL probes against the raw DB.
                     Classify tables: active vs empty vs system noise vs analytically relevant.
                     Document column fill rates, ID formats, cross-system key types.
                     Identify data quality traps before they reach a dashboard.

Step 2: STAGE        Write one staging model per analytically relevant table.
                     Clean names, cast types, rename misleading columns.
                     Add YAML documentation for every column.
                     Run not_null / unique / accepted_values tests.
                     Materialized as views — no storage cost, always fresh.

Step 3: INTEGRATE    Build intermediate models that apply business logic.
                     Resolve foreign keys to human-readable names.
                     Derive computed flags and metrics (is_done, cycle_time_days).
                     Enrich from secondary sources (description parse, external seeds).
                     Keep each intermediate model at a single grain.

Step 4: MART         Build one fact or dimension table per business domain.
                     Validate grain (COUNT(*) = COUNT(DISTINCT pk) — always).
                     Run the full test suite: 0 ERROR before publishing.
                     Document every column with semantic meaning and caveats.
                     Materialized as table — fast for dashboards.

Step 5: DASHBOARD    Point all BI tools at the mart layer only.
                     No BI tool ever queries a raw operational table.
                     KPI definitions are in the mart, not in the dashboard.
                     New panels are SQL on the mart — no business logic in Grafana.
```

### Next Domains Already Scoped (from `domain_coverage_plan.md`)

| Domain | Key Tables | Key Metric Unlocked | Status |
|---|---|---|---|
| **Materials / Cost** | `maintenance_material_line`, `stock_valuation_layer` | Real material cost per work order | Deferred (reliable cost needs SVL join) |
| **Finance / Revenue** | `account_move`, `account_move_line` (85,821 rows) | Revenue recognition, AR reconciliation | Staged but not yet modeled |
| **HR Attendance** | `hr_attendance_zk_temp` (141,601 rows) | Technician attendance vs dispatch | 92% of Mar 2026 unprocessed — quality blocker documented |
| **Inventory / Procurement** | `stock_move`, `stock_picking`, `approval_request` | Material issuance vs consumption | Disconnected from work orders — needs bridge model |
| **CRM Sales Pipeline** | `crm_lead` (7,375 rows) | Installation conversion funnel | Not yet staged |

Each domain follows the same five steps. The first domain (`fct_field_service_work_orders`) proved the pattern. The remaining domains benefit from the foundation already built.

---

## Supporting Evidence

All findings referenced in this document are sourced from the following workspace files:

| File | What It Proves |
|---|---|
| `PROJECT_FULL_STORY.md` | End-to-end narrative: probing → staging → intermediate → mart |
| `FACT_TABLE_VALIDATION_REVIEW.md` | Grain, status flags, test suite, coverage per column type |
| `COMPARATIVE_ANALYSIS_FEEDBACK.md` | Direct comparison of other engineers' SQL vs dbt outputs |
| `OTHER_ENG_SQL_EVALUATION.md` | Query-by-query evaluation of raw SQL failures |
| `EXTERNAL_SYSTEMS_LINKAGE_PROOF.md` | Cross-system architecture proof with column-level evidence |
| `RES_PARTNER_PROOF.md` | res_partner population breakdown; subscriber count inflation |
| `CASE_TRACE_PROOF.md` | End-to-end trace of work order 3231 |
| `docs/dbt_architecture/grafana_dashboard_queries.md` | 42 Grafana panels — all reading from the dbt mart |
| `docs/dbt_architecture/description_parse_findings.md` | FAT coverage: 24.7% (FK) → 96.7% (description parse) |
| `docs/dbt_architecture/cross_system_architecture.md` | sap_id naming trap; FMS / Creatio / HRIS architecture |
| `docs/dbt_architecture/domain_coverage_plan.md` | Scoping 1,027 tables into 5 staged domains |
| `findings/database_discovery_report.md` | Full DB discovery: table counts, size, fill rates |

---

*This document was produced from systematic live database probes and validated dbt build outputs. Every figure is directly queryable from `dbt_dev_marts.fct_field_service_work_orders` or from the cited probe files. No number in this document is estimated.*
