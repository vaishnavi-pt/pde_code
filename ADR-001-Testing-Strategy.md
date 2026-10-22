
ADR-001: End-to-End Testing Strategy and Tooling for PDE MVP Pilot
Status: Proposed / Under Review
Context: Preclinical Data Environment (PDE) MVP Pilot Implementation (1–3 prioritized DMPK assays)
Author: Lead QA & DQ Test Engineer
Timebox: 2 Days
1. Context & Problem Statement
For the initial MVP release of the Bayer Preclinical Data Environment (PDE) pilot, we need an agreed-upon, pragmatic testing strategy that ensures high data quality, robust pipeline reliability, and smooth integration across upstream labs (e.g., LabVantage/RedLims), cloud data storage (Databricks/Delta Lake), and downstream consumers (e.g., PIx5, Excel connectors).

Given the 2-day timebox and pilot nature (focusing on 1–3 DMPK assays, GLP/Part 11 full compliance being out-of-scope for the pilot itself), we must balance thorough quality coverage with rapid MVP delivery speed.

2. Decision Drivers
Data Quality & Integrity: Assays require strict adherence to data contracts (JSON/Parquet), validation rules, and automated DQ feedback loops.
E2E Traceability: Ability to track data seamlessly from ingestion (Bronze) through validation (Silver) to curated publication (Gold/PIx5).
Ecosystem Compatibility: Integration testing across Excel templates, REST APIs, and Databricks endpoints.
Efficiency & Sustainability: Leveraging Bayer and EPAM standard tech stacks (GitHub Actions, Python pytest, etc.) to minimize custom tooling overhead.
3. Considered Options
Option A: Manual-Heavy Testing Approach. Rely primarily on manual UAT and spot-checks by data engineers and scientists. (Rejected: High risk of data type corruption, slow feedback loop, fails to scale for automated DQ rules).
Option B: Comprehensive Automated Pyramid (Pytest + Great Expectations + GitHub Actions). Structured automated testing across unit, integration, and E2E layers integrated into CI/CD. (Selected).
4. Decision Outcome: The Testing Strategy
We adopt a Test Automation Pyramid tailored for a data-centric cloud platform, split into three clear tiers:

Level 1: Unit Testing (Target Coverage: >80% for custom code/connectors)
Scope: Individual components, Python transformation scripts, custom validation logic, schema contract parsers, and Excel connector sub-routines.
Tooling: pytest (Python framework).
Execution: Triggered automatically on every pull request via GitHub Actions.
Level 2: Integration & Data Quality (DQ) Testing
Scope:
API endpoints, SQL queries, and Databricks delta lake transformations.
Validation of automated DQ rules (e.g., checking that numeric ranges, mandatory fields, and compound IDs match expected schemas).
Tooling:
pytest for pipeline integration tests.
Great Expectations (or built-in Delta Live Table expectations) for programmatic data quality assertions across Bronze 
→
 Silver 
→
 Gold layers.
Level 3: End-to-End (E2E) Functional & Pipeline Testing
Scope: Complete pipeline execution for the 1–3 prioritized DMPK assays:
Upload (via Excel/API) 
→
 Validation 
→
 Quality-State Assignment 
→
 Retrieval (PIx5 / Excel).
Role-Based Access Control (RBAC) verification across distinct user groups.
Tooling: Automated API/pipeline integration test scripts running against a dedicated test environment.
UAT Alignment: EPAM will execute a one-time preliminary E2E run in the test environment to flush out bugs before handing off formal test cases to Bayer for User Acceptance Testing.
5. Automation Strategy & Toolchain Summary


| Test Level | Target Automation Level | Primary Framework / Tool | Execution Trigger |
| :-- | :-- | :-- | :-- |
| Unit Testing | 80%+ code coverage | pytest (Python) | PR Merge / GitHub Actions CI |
| Data Quality (DQ) Rules | 100% automated assertions | Great Expectations / Databricks DLT | Ingestion Pipeline / Nightly CI |
| Integration Testing | Key integration paths automated | pytest, Postman/Newman (for APIs) | Post-Deploy / Nightly |
| E2E Pilot Validation | Core happy path + critical negative paths automated; manual exploratory UAT | Custom Python E2E test suites + Bayer UAT scripts | Pre-release Milestone Gate |

6. Consequences & Quality Gates
Positive Consequences:
Clear, repeatable quality gates prevent bad schema changes from breaking downstream models or Excel connectors.
Automated DQ checks provide immediate feedback to assay submitters.
Negative Consequences / Trade-offs:
Requires initial setup time during sprint 1 to configure Great Expectations and test data fixtures.
Definition of Done (DoD) Quality Gate for Releases:
All unit tests pass with required coverage.
Zero critical or high-severity automated DQ failures on the pilot DMPK assays.
Written UAT sign-off from Bayer business/IT stakeholders.