# Build-Out

Working material for the Workstream 04 (Data Foundation Build) build-out. This folder holds vendor-authored plans and build-phase assets. Nothing here is normative: [docs/](../docs/index.md) is the lasting rulebook for the Rubicon team, and where a document in this folder conflicts with docs/, docs/ wins. Conflicts are decided and recorded in the [platform decision register](../docs/platform/decision-register.md).

## Contents

- [nimblegravity/](nimblegravity/) - Nimble Gravity's WS04 documents, vendor-authored drafts under joint review (NG lead: Dave Newman; client counterpart: Graham Lammers):
  - [Architecture & Implementation Plan](nimblegravity/ws04-architecture-and-implementation-plan.md) - target lakehouse architecture, ingestion lanes, environment topology, delivery phases
  - [Azure Platform Plan](nimblegravity/ws04-azure-platform-plan.md) - subscription/RG topology, network design, resource inventory, identities, private endpoints
  - [BRD / Tech Spec](nimblegravity/ws04-brd-tech-spec.md) - business context, scope, stakeholders, risks, deliverables

These are drafts under joint review: client-side revisions land by PR and are fed back to NG. Never use an edit here to record a decision; decisions land in the register and the owning doc in docs/.

## How NG proposals map to the platform standards

The 2026-08 review decided the conflicts between NG's plans and this repo (register IDs in parentheses). When building, follow the docs/ column, not the vendor document.

| NG proposal topic | Governing doc | Decision |
| --- | --- | --- |
| Compute network model (VNet injection + serverless/NCC) | [Azure infrastructure](../docs/platform/azure-infrastructure.md), [Compute policies](../docs/platform/databricks-compute-policies.md) | Hybrid model adopted (D1) |
| Three workspaces, subscription per environment | [Environments](../docs/platform/databricks-environments.md) | Superseded by the repo standard: two workspace tiers, env-per-catalog (D2) |
| Management groups (single custom group, defer CAF hierarchy) | [Azure infrastructure](../docs/platform/azure-infrastructure.md) | Adopted (D8) |
| Metastore per environment | [Azure infrastructure](../docs/platform/azure-infrastructure.md) | Superseded by the repo standard: one regional metastore, a platform limit (D3) |
| Naming (`sp-ng-*`, `rg-rubicon-<env>-<workload>`, `sub-rubicon-*`) | [Naming conventions](../docs/platform/naming-conventions.md) | Superseded by the repo conventions; vendor never in names (D4/D5) |
| Storage layout and containers | [Azure infrastructure](../docs/platform/azure-infrastructure.md), [Medallion data practices](../docs/practices/medallion-data-practices.md) | Container-per-layer external locations adopted (D6) |
| Private endpoint matrix | [Azure infrastructure](../docs/platform/azure-infrastructure.md) | Adopted per tier alongside NCC (D7) |
| CI/CD identities and OIDC | [GitHub CI/CD and deployment](../docs/platform/github-cicd-and-deployment.md), [Service principal auth](../docs/platform/databricks-service-principal-auth.md) | Follow repo standards; NG identity names superseded under D4/D5 |
| Genie / gold-layer consumption | [Genie spaces](../docs/platform/databricks-genie-spaces.md), [Metric views](../docs/platform/databricks-metric-views.md) | Follow repo standards |

## Vendor-driven work items

These items originate in NG's WS04 documents and are NG-driven; they are tracked here for visibility, but any resolution that touches platform standards lands as a PR to docs/, not as a vendor document. Open items with owners live in the [decision register](../docs/platform/decision-register.md#open-items).

- **Source system inventory** - full inventory including GUS and other operational/reporting systems; confirmed so far: Azure SQL DB, Amazon RDS, Amazon S3 pending the Palantir Foundry investigation.
- **Source onboarding sequence** - priority order for onboarding sources within the Build phase.
- **AWS environment handoff** - documented handoff of AWS/Terraform knowledge from the exiting vendor (Rackspace); gating dependency for migration steps.
- **AWS-to-Azure connectivity model** - VPN/ExpressRoute vs public endpoint with allowlist vs cross-cloud PrivateLink vs landing data behind an Azure endpoint, for RDS/S3 ([register open item](../docs/platform/decision-register.md#open-items)).
- **Azure SQL DB network location** - where the existing source lives and its reconciliation into the hub-and-spoke topology ([register open item](../docs/platform/decision-register.md#open-items)).
- **IP address plan validation** - hub and spoke CIDRs against AWS VPC ranges and existing Rubicon networks ([register open item](../docs/platform/decision-register.md#open-items)).
- **Azure region confirmation** - pending tenant review ([register open item](../docs/platform/decision-register.md#open-items)).
- **OneLake mirroring evaluation** - for the Power BI connection to Gold.
- **Data quality and reconciliation thresholds** - tolerance thresholds co-defined with Rubicon business stakeholders during the Build-to-Certify transition.
