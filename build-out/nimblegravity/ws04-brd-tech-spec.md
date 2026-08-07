**NIMBLE GRAVITY**

Rubicon Technologies Transformation Program

**Workstream 04: Data Foundation Build**

*Business Requirements Document & Technical Specification (Draft for
Review)*

| **Document Control**    |                                                                                                               |
|-------------------------|---------------------------------------------------------------------------------------------------------------|
| **Program**             | Rubicon Technologies Transformation Program                                                                   |
| **Client**              | Rubicon Global, LLC                                                                                           |
| **Workstream**          | 04: Data Foundation Build                                                                                     |
| **Nimble Gravity Lead** | Dave Newman                                                                                                   |
| **Client Counterpart**  | Graham Lammers, Sr. Director, Data                                                                            |
| **Governing agreement** | Statement of Work: Data Foundation Build, under the Master Professional Services Agreement dated June 9, 2026 |
| **Document status**     | Draft for review, prepared August 6, 2026                                                                     |
| **Companion document**  | WS04 Architecture & Implementation Plan (full technical detail)                                               |

# **1. Purpose of This Document**

This is the workstream-level BRD and technical specification for Data
Foundation Build, following the same structure used across the program's
other workstream documents. It sits underneath the Program Onboarding &
Kickoff Guide, which covers cross-workstream context, and above the
companion Architecture & Implementation Plan, which covers technical
detail. Read this for the business case, scope, and delivery model; read
the companion document for the architecture itself.

# **2. Business Context**

Rubicon's core operating platform, GUS, carries data debt that directly
limits reporting and downstream automation: roughly 30% of tables have
no single-field primary key, the central work-order table has no
reliable timestamp watermark, and records deleted in GUS disappear from
reporting permanently. There has never been a formal reconciliation
between GUS and Business Central. The financial estimate table that
drives accrual reporting runs to roughly 30 million rows, takes hours to
refresh, and has no formally defined business rules behind it.

These details shape the architecture directly: an ingestion approach
that can capture change (including deletes) before it's lost, a bronze
layer that preserves raw history rather than only current state, and
silver-layer logic that can absorb missing primary keys and undocumented
business rules without propagating errors into gold.

Reporting today runs on an existing Azure SQL Server-based warehouse
with stored-procedure-driven logic and no clear ownership model. This
workstream replaces that foundation with a governed lakehouse, and the
Reporting Certification + Cutover workstream (WS03) depends on this
build to move reporting onto a reliable base.

# **3. Scope**

## **In Scope**

- Lakehouse architecture setup and configuration on the target platform
  (Databricks on Azure)

- ETL pipeline development for the mutually agreed priority source
  systems, expected to include order-to-cash, reporting, finance, and
  operational data sources prioritized during onboarding

- Data quality validation and reconciliation against mutually agreed
  source data and business rules

- Reporting enablement support: foundational data model and semantic
  layer inputs, validation datasets, and sample reporting views needed
  to confirm the foundation works

- Performance tuning and monitoring setup prior to handoff

## **Out of Scope**

- Source system remediation or data cleansing within systems of record
  (Client-owned, e.g. fixing GUS data quality at the source)

- Adding source systems materially beyond the mutually agreed source
  list, unless added through change order

- Report and dashboard development beyond what's needed to validate the
  foundation (covered under the Reporting Certification workstream)

Any change to this scope goes through a mutually agreed change order,
consistent with the SOW.

# **4. Objectives and Success Criteria**

**Program-level objective this workstream serves:** build a reliable,
governed data foundation to replace the ad hoc, stored-procedure-driven
reporting described in Section 2.

**Workstream-level success criteria:**

- Configured lakehouse environment with documented architecture

- ETL pipelines live for the agreed, in-scope source systems, with the
  source list and priority sequence documented at onboarding

- Data quality and reconciliation validation completed against agreed
  source data and business rules, including agreed certification
  criteria and exceptions

- Stabilized environment with monitoring in place, ready for handoff to
  Rubicon or Rubicon-designated run-state support

# **5. Stakeholders**

| **Name**           | **Organization** | **Role**                  | **Why They Matter to This Workstream**                                                |
|--------------------|------------------|---------------------------|---------------------------------------------------------------------------------------|
| Graham Lammers     | Rubicon          | Sr. Director, Data        | Primary client counterpart for Data Foundation Build                                  |
| Robert "Rob" Smith | Rubicon          | Transformation Lead       | Primary client contact; owns board-level commitments                                  |
| Jesus "Beto" Sainz | Rubicon          | Enterprise Architect      | Primary holder of GUS architecture knowledge relevant to source access and data model |
| Dave Newman        | Nimble Gravity   | Workstream Lead           | Owns delivery of this workstream                                                      |
| Ygor Tazinaffo     | Nimble Gravity   | Client Engagement Manager | Client relationship owner                                                             |
| Jose Paz           | Nimble Gravity   | Director of Delivery      | Overall delivery accountability                                                       |
| Mauro Lopez        | Nimble Gravity   | VP of Engineering         | Overall technical accountability                                                      |

# **6. Architecture Summary**

The target architecture is a Databricks lakehouse on Azure, using a
metadata-driven medallion pattern (bronze, silver, gold) built with
Spark Declarative Pipelines and deployed through Declarative Automation
Bundles. Four ingestion lanes handle change data capture, batch
structured, batch unstructured, and streaming (the last one architected
as a future placeholder, with no streaming source identified today).
Unity Catalog is the single governance layer across all environments.
Environment topology uses one Azure subscription and one Databricks
workspace per environment (dev, test, prod) under a single Azure tenant.

Full detail, including the ingestion diagram, environment topology,
CI/CD approach, and open kickoff decisions, is in the companion **WS04
Architecture & Implementation Plan**.

# **7. Governance and Cadence**

This workstream follows the program-level governance cadence set out in
the Program Onboarding & Kickoff Guide: weekly delivery status, weekly
leadership status, weekly workstream status, and daily workstream
standups. Exact recurrence is still being finalized with Rubicon.

Workstream-specific governance: architecture and platform decisions are
tracked as Unity Catalog-governed configuration and
infrastructure-as-code, not informal agreement. Any deviation from the
architecture in the companion document should be raised in a formal
conversation before being implemented.

# **8. Operating Model**

This workstream operates inside the program's shared operating model: a
single cross-workstream dependency tracker, a shared RAID log, and this
BRD/Tech Spec as the workstream-specific governing document.

Two cross-workstream dependencies are known, with more potentially added
in the future subject to discovery:

- **WS02 (GUS Maintenance + Knowledge Transfer):** source access and GUS
  data model knowledge needed for this workstream's Lane 1/2 ingestion
  design flows through WS02.

- **WS03 (Reporting Certification + Cutover):** consumes this
  workstream's gold layer output; sequencing between the two needs to be
  coordinated so WS03 isn't blocked waiting on a source this workstream
  hasn't onboarded yet.

# **9. Timeline**

| **Phase** | **Focus**                                                      | **Estimated Timing** |
|-----------|----------------------------------------------------------------|----------------------|
| Build     | Lakehouse setup, ETL pipeline development, source integration  | Months 1-4           |
| Certify   | Data quality validation, reconciliation against source systems | Months 4-5           |
| Stabilize | Performance tuning, monitoring, handoff to run-state support   | Month 6              |

Detailed sequencing within the Build phase is a live open item for
kickoff and is intentionally left as a placeholder here and in the
architecture document. It will be filled in once the source inventory
and priority order are confirmed.

# **10. Risks**

| **Risk**                           | **Detail**                                                                                                                            | **Mitigation**                                                                                                                                                             |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AWS environment handoff            | Current infrastructure knowledge (AWS, Terraform) sits primarily with an exiting vendor team, with minimal documentation left behind. | Program-level guardrail requires a documented environment handoff before any Azure migration step proceeds. Treat this as a gating dependency, not a parallel task.        |
| Incomplete source inventory        | Only Azure SQL DB, RDS, and S3 are confirmed; GUS and other systems are not yet mapped.                                               | Confirm at kickoff; architecture is designed to absorb additional sources without redesign.                                                                                |
| Undefined certification tolerances | Data quality and reconciliation thresholds aren't set yet.                                                                            | Define collaboratively with Rubicon business users during the Build-to-Certify transition, per the SOW.                                                                    |
| Source data quality (GUS)          | Missing primary keys, no timestamp watermark on core tables, deleted records vanishing from reporting.                                | Architecture accounts for this (Section 2, Section 6), but the underlying data debt remains Rubicon's to remediate at the source; this workstream does not fix GUS itself. |
| Cross-workstream sequencing        | WS03 depends on this workstream's gold layer; this workstream depends on WS02 for source access and data model knowledge.             | Track explicitly in the shared dependency tracker (Section 8).                                                                                                             |
| Infrastructure maturity            | Rubicon's existing Azure environment may need enhancement to align with a well-architected framework.                                 | Nimble Gravity to support this effort.                                                                                                                                     |

# **11. Deliverables**

1\. Configured lakehouse environment with documented architecture

2\. ETL pipelines for the agreed, in-scope source systems, with the
initial source list and priority sequence documented during onboarding

3\. Data quality and reconciliation validation report against agreed
source data and business rules, including agreed certification criteria
and exceptions

4\. Stabilized environment with agreed monitoring in place, ready for
handoff to Rubicon or Rubicon-designated run-state support

# **12. Onboarding Checklist (Workstream 04 Specific)**

- Read this document and the companion Architecture & Implementation
  Plan

- Confirm access requests and/or role provisioning are in for: Azure
  tenant/subscriptions (once provisioned), GitHub repositories,
  Databricks workspace (once provisioned)

- Confirm access to or obtain current AWS environment documentation

- Review the cross-workstream dependency tracker for WS02 and WS03
  linkages

- Confirm source systems and initial priority order
