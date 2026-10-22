# ADR-001: Testing Strategy for the PDE MVP (First E2E Release)

| | |
|---|---|
| **Status** | Proposed (draft for review by Tech Lead, DevOps Engineer, FAIR Engineer, Data Governance Lead, Bayer SPOC) |
| **Date** | 2026-10-21 |
| **Author** | Key Quality Engineer / QA & Data QA Lead |
| **Scope** | PDE Pilot: up to 3 DMPK (in vitro ADME) assays, RedLIMS/LabVantage to Databricks (bronze/silver/gold) to EvaluWhale to PIx5/C360, Excel connector, SQL/API access |

## 1. Introduction

- The PDE MVP is a time-boxed pilot (8-9 weeks implementation + 3 weeks UAT) for up to three DMPK assays. Acceptance (SoW 2.b) requires: E2E flow upload to validation to quality-state to gold/PIx5; role-based access verified for at least two user groups; at least one upstream and one downstream integration validated (LabVantage, EvaluWhale, DMPK Excel); reproducible access via Excel/API/Databricks; measurable DQ improvement with a usable feedback loop; written UAT sign-off.
- Quality is mostly **data quality**, not UI behaviour. The risk sits in pipelines, rules, integrations, access control and traceability.
- Constraints: no PII; production data must not leave Bayer infrastructure; GLP/21 CFR Part 11 compliance is out of scope, but data integrity must be demonstrable (ALCOA+) so a later validation is not blocked; deployment to production is done by Bayer; UAT is happy-path for up to 3 assays; Bayer feedback turnaround is short (deemed acceptance), so evidence must be clear and fast to review.
- Bayer standard toolchain: GitHub + GitHub Actions, ADO (backlog, Azure Test Plans), Confluence, Terraform, Datadog, Entra ID/IDNow. Databricks with Unity Catalog on AWS (SMART Cloud).

## 2. Test deliverables

### 2.1 Test levels and expected depth

We adopt a **test pyramid weighted toward data and contract tests**, with a thin E2E layer.

| Level | What is covered | Depth for MVP | Owner |
|---|---|---|---|
| **Unit** | Transformation logic (bronze to silver to gold), DQ rule functions, FAIR identifier generation, API handlers, schema/data-contract validators | Target >= 80% line coverage on custom code; 100% of DQ rules have positive and negative cases | Developers (QA reviews, defines minimum cases) |
| **Data quality / data contract** | Schema, mandatory fields, ranges, enumerations, referential integrity (compound/project/assay IDs), reference-compound pass/fail, expected variability, completeness; per layer | Every assay-specific rule from AssayLink mapping covered with golden and bad datasets; checks run in pipeline and in CI | QA + Data Governance Lead + DMPK SMEs |
| **Integration** | RedLIMS export/ingestion, file upload endpoint, AssayLink, EvaluWhale (bidirectional), C360/PIx5 publication, API Gateway, Databricks tables/views/SQL endpoint, Excel connector, Entra ID/IDNow SSO | Each integration: contract test + one real connectivity test in test env, including error paths (timeout, malformed payload, auth failure) | QA + Integration Coordinator |
| **Security / access** | RBAC across at least 2 groups, quality-state gating (analysts see bronze/silver; consumers gold only), dataset/assay-level segregation, audit logging | Full permission matrix, automated; negative tests mandatory | QA + IAM Lead |
| **E2E** | Upload to validation to quality state to sign-off to publication to retrieval (SQL/API/Excel), per assay (3 assays) | Happy path automated + key negative paths (rejected upload, correction/versioning, silver-to-gold rejection) | QA |
| **Traceability / audit** | Provenance (source, user, timestamp, internal/external), version history, reason codes, reconstructable lineage for a sampled set of records | Audit reconstruction test: pick N records, rebuild lineage end-to-end from stored metadata | QA |
| **FAIR conformance** | Persistent IDs resolve, metadata completeness, catalog entries, machine-readable contracts | Checklist + automated metadata completeness checks, jointly with FAIR Engineer | QA + FAIR Engineer |
| **Non-functional (light)** | Ingestion of representative file sizes, API response sanity, pipeline rerun/idempotency | Smoke-level only; no formal load/DR testing (DR is out of scope) | QA + DevOps |
| **UAT** | 4 personas (lab scientist, lab head, project lead, data scientist), UC1-UC3 | Scripted happy path, 3 assays, max 3 weeks, structured defect log | QA leads, Bayer users execute |

### 2.1.1 PDE data-quality verification

The checks below turn the selected controls in `data_quality_pde.md` into verifiable test outcomes. Tests use synthetic or de-identified fixtures and run against the dedicated Databricks test environment where Delta behavior is required.

| Dimension / control | Layer and test | Expected result |
|---|---|---|
| **Consistency — optimistic concurrency** | Integration test concurrent reads and writes on representative Delta tables; force a conflicting write where feasible | Readers observe a complete committed version, never staged files or a partial commit. A conflicting writer fails without changing the current version and is retried against the latest version by the job. |
| **Accuracy — `NOT NULL` and `CHECK`** | For each Silver/Gold constraint, test valid rows and boundary/invalid rows, including null business keys, out-of-range values, and cross-column rules such as end before start | Valid rows commit. A constraint violation rejects the write and leaves the table at its previous version. Rules requiring another table are tested in the Silver job and must block its commit when they fail. |
| **Accuracy — `RESTORE` and `VACUUM`** | Operational recovery exercise in a disposable test table: commit a known bad version, restore the last good version, then verify the current rows and version history. Separately validate the scheduled retention policy and the manual early-vacuum approval/runbook without vacuuming protected test history. | Restore returns the affected table to the selected good version; the recovery is recorded. Weekly vacuum respects the 365-day retention window. Early vacuum is an explicitly approved recovery/retention action, not part of a normal load, and the runbook records that removed versions cannot be restored. |
| **Validity — schema enforcement** | Attempt writes with a type conflict, an unapproved extra column, and a missing nullable/mandatory column | Conflicting or unapproved schemas fail without changing the table. A missing nullable column is written as null; a missing mandatory column is rejected by its constraint. |
| **Validity — schema evolution and overwrite** | Test an approved additive Bronze write with per-write `mergeSchema`; verify Silver/Gold remain unchanged until their contracts are updated. Separately test an approved breaking change with `overwriteSchema` on a disposable table. | Only the approved Bronze write gains additive columns; session-wide auto-merge remains off. Overwrite replaces the schema atomically and the previous version remains time-travelable until retention cleanup. Routine loads do not use `overwriteSchema`. |
| **Completeness — atomicity and provenance** | Inject a failure during a table write and during a grouped publication; inspect table versions and source-file metadata. Check catalog comments and required table properties. | A table commit is all-or-nothing. No table in a grouped publication is exposed as the new release if a blocking check fails; verify the configured orchestration/publication mechanism enforces this across tables. Every Bronze record can be traced to its input filename, and required table descriptions and layer/source/feed tags are present. |
| **Timeliness — freshness and shared-table writes** | Load representative concurrent feeds and compare file landing / `extract_as_of` with the committed Bronze version against the agreed per-feed freshness target | Each feed meets its agreed freshness target. A concurrency conflict is retried and eventually processed or reported as a failed job; no feed is silently dropped. |
| **Uniqueness — Silver business key** | Replay an identical file and load overlapping input; compare duplicate handling at the `_bk` key and verify Silver output | Exact duplicate copies collapse to one Silver row per `_bk`. The test does not assume `dropDuplicates` selects the newest record: inputs with differing values for the same key must be handled by the specified ranking/business rule or fail validation, never pass based on arbitrary Spark row selection. |

For every failure test, capture the input/run identifier, expected and observed outcome, table version before and after, job result, and actionable error or DQ feedback. Verify a rerun from landing files or the last good version is safe. Use `RESTORE` only to repair already-committed bad data; it is not a successful-load step. `VACUUM` is a separate scheduled workflow and must not run as part of routine ingestion or recovery verification.

Out of MVP testing scope: formal GxP/Part 11 validation (IQ/OQ/PQ), DR/failover, CRO onboarding, instrument-level data, full performance/load testing. These are recorded as post-pilot recommendations in the final test report.

### 2.2 Level of automation

Principle: **automate everything that repeats on every deployment; keep humans for judgment.**

- **Automated (target 85-90% of test cases):** unit, DQ/data contract, API and integration contract tests, Delta constraint/schema/uniqueness assertions, RBAC matrix, pipeline E2E regression for the 3 assays, lineage/provenance assertions, FAIR metadata completeness, infrastructure/IaC policy checks, and non-destructive retention-policy checks.
- **Semi-automated:** Excel connector (scripted where feasible on the Bayer physical laptop; manual verification of VBA-template behaviour).
- **Manual / controlled operational exercise:** UAT execution by business users, exploratory testing, usability of the validation feedback, silver-to-gold sign-off experience, and `RESTORE` / early-`VACUUM` runbook verification in a disposable test table. Destructive retention actions are never automated against protected or production data.

### 2.3 Frameworks and tooling (proposed)

| Area | Choice | Rationale / note |
|---|---|---|
| Test runner (Python) | **pytest** (+ pytest-cov) | De facto standard for pipelines and APIs |
| Pipeline/data tests | **Databricks DQX** for data quality rules that must be authored, reviewed, and reported by data stewards; **Databricks DLT / Lakeflow expectations** for in-pipeline enforcement | Rules are versioned in Git, linked to AssayLink model, and executed through the Databricks-native quality framework to minimize custom tooling and keep rule governance consistent |
| Databricks testing | `databricks-connect` / Databricks Asset Bundles with test jobs in a dedicated test workspace; small synthetic and de-identified assay datasets | Avoids copying production data off Bayer infra |
| Contract testing | **JSON Schema / Parquet schema checks** for data contracts; **Pact** (or OpenAPI-based validation with Schemathesis) for API Gateway endpoints | Matches "machine-readable data contracts" in scope |
| API testing | **Postman/Newman** or pytest + requests; OpenAPI as source of truth | Newman is easy to run in GitHub Actions |
| BDD / E2E | **Vividus** (already in Bayer DevOps toolset), fallback pytest-bdd | Gherkin scenarios readable by DMPK SMEs; reuse for UAT scripts |
| Test management | **ADO Test Plans** (cases, runs, traceability to requirements and bugs) | Single source of evidence for Bayer |
| CI/CD | **GitHub Actions** | Bayer standard |
| IaC / security | Terraform validate/plan + **Checkov/tfsec**; SAST/SCA (GitHub Advanced Security or Bayer standard) | Part of quality gates |
| Observability | Datadog / CloudWatch dashboards for pipeline failures and DQ metrics | DQ KPI reporting |
| Test data | Synthetic + de-identified "golden" datasets per assay (clearance, CYP DDI), versioned in Git (no production data, no PII) | Compliance with SoW data assumptions |

### 2.4 Quality gates in CI/CD

| Gate | Trigger | Criteria to pass |
|---|---|---|
| G1 Pull request | Every PR | Lint, unit tests, coverage threshold, SAST/secret scan, positive and negative tests for changed DQ rules, schema-contract checks, IaC policy checks |
| G2 Merge to main / deploy to DEV | Merge | G1 + integration contract tests + Delta schema-enforcement/evolution checks + constraints and uniqueness tests |
| G3 Deploy to TEST | Release candidate | G2 + automated E2E for 3 assays + RBAC matrix + lineage/provenance and catalog-metadata assertions + freshness checks + FAIR completeness; run concurrency and grouped-publication failure tests |
| G4 UAT entry | Before UAT | 0 open Critical/High defects, all G3 green, test evidence report published |
| G5 Handover to Bayer for production | After UAT | Written UAT sign-off, defect status accepted by Bayer |

Defect severity: Critical (data corruption, wrong data in gold, access breach), High (rule not enforced, integration broken), Medium, Low. Exit criteria for MVP release: no open Critical/High; Medium accepted with documented workaround.

Delta recovery tests are run only in an isolated, disposable test table. Do not disable the configured retention safety checks or issue an early `VACUUM` against shared, protected, or production tables to satisfy a test. Record the table/version and approval for any controlled destructive exercise.

### 2.5 Data quality KPIs and pilot success metrics

- Rule coverage: % of AssayLink-defined rules implemented and tested
- First-pass ingestion success rate and rejection rate, with top violated rules
- Time from upload to feedback to submitter
- Completeness of mandatory metadata (target 100% in gold)
- % of gold records with full provenance (target 100%)
- Silver-to-gold cycle time
- Defect counts by severity, escape rate, mean time to resolve
- Baseline vs. pilot DQ comparison (acceptance criterion: measurable improvement)

Reported fortnightly alongside the status meeting.

### 2.6 Data integrity and traceability approach

Test against **ALCOA+** attributes: attributable (user/system), contemporaneous (timestamps), original (immutable bronze), accurate (rules), complete, consistent, enduring, available. Test evidence is stored in ADO Test Plans and the Confluence hub with run IDs, commit SHAs and environment versions so any result can be reproduced.

## 3. Alternatives considered

1. **Manual-heavy testing:** rejected, too slow for an 8-9 week build and not repeatable per deployment.
2. **Full E2E-only testing:** rejected, brittle and slow, poor fault isolation across many integrations.
3. **Heavy upfront GxP validation:** rejected, explicitly out of scope; we prepare evidence structure instead.
4. **Single DQ framework mandated everywhere:** adoption of **Databricks DQX** as the standard PDE DQ framework, with Databricks-native expectations used for in-pipeline enforcement where required.

## 4. Consequences

- (+) Fast feedback, repeatable regression for the three assays, audit-ready evidence, DQ rules testable by non-developers.
- (+) Aligns with Bayer-standard tooling, lowering access and approval friction.
- (-) Upfront effort to create golden datasets and DQ rule test cases; requires SME time in the first sprints.
- (-) Excel/VBA connector cannot be fully automated; depends on a Bayer laptop being available.
- (-) Dependence on Bayer test environments and external system test instances (RedLIMS, EvaluWhale, C360, AssayLink); instability there impacts E2E reliability.

## 5. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Test environment or access delayed (2-week provisioning assumption) | Start with unit/DQ tests locally on synthetic data; request test instance access now |
| No stable test instances for external systems | Contract tests plus mocks (WireMock/Mockoon) for CI; real integration tests only in TEST env |
| Insufficient or sensitive test data | Co-create golden datasets with DMPK SMEs in sprint 1; no production data |
| DQ rules change late (AssayLink mapping evolves) | Rules versioned and tied to schema versions; regression pack re-run on change |
| Short UAT window and 2-day feedback turnaround | Pre-UAT dry run, scripts in Gherkin, daily defect triage |
| Unknown PDC behaviour | Add compatibility checks in discovery; log as risk |

## 6. Open points to confirm (before moving to Accepted)

1. Databricks DQX configuration and governance model for PDE: platform owners to confirm rule ownership, reporting, and enforcement scope.
2. Are test and UAT environments separate from DEV? Which external systems offer test instances?
3. Is Vividus approved and available for this project, or do we use pytest-bdd?
4. Coverage thresholds and the severity model: confirm with Bayer SPOC.
5. Who from Bayer executes UAT (names, capacity), and are external collaborators in the MVP UAT or deferred?
6. Which environment hosts audit/lineage reconstruction tests, and what retention applies to test evidence?

## 7. Next steps

1. Review this ADR with Tech Lead, DevOps, FAIR Engineer, Data Governance Lead (day 2).
2. Publish as ADR-001 in the documentation hub; link from ADO item.
3. Create the master test plan, ADO Test Plan structure, and golden datasets backlog items.
4. Draft CI quality gates with DevOps.