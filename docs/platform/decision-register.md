# Platform Decision Register

Dated record of platform architecture decisions and the still-open items, in one place. Each decision names the doc that now carries the normative content; this register is the index, never the rulebook. A decision made anywhere else (a vendor document, a meeting, a chat) is not made until it lands here and in its owning doc via PR.

Context for the 2026-08 entries: Workstream 04 (Data Foundation Build) runs with a vendor, Nimble Gravity (NG). NG's Architecture & Implementation Plan and Azure Platform Plan diverged from this repo in several places; the platform owner decided the conflicts and this register records the outcomes. Vendor-driven open items are outlined in the repo [README](../../README.md).

## What this covers

- Decided items with date, rationale, and the owning doc
- Open items with owners
- The rule for recording future decisions

## Decisions

| ID | Date | Decision | Rationale | Normative doc |
| --- | --- | --- | --- | --- |
| D1 | 2026-08 | Hybrid compute model: workspaces are VNet-injected with serverless enabled via NCC; serverless stays the preferred default; no separate fallback workspace. Hub-and-spoke network with forced egress through the hub firewall. | CDC/JDBC pull connections to source databases (Azure SQL DB, Amazon RDS) need a direct, controlled network path serverless cannot provide. | [Azure infrastructure](azure-infrastructure.md), [Compute policies](databricks-compute-policies.md) |
| D2 | 2026-08 | Two workspace tiers, environments as catalogs, promotion by bundle-target redeploy of the same commit SHA. Confirmed against a vendor-proposed three-workspace, subscription-per-environment design, which is rejected. Subscription topology stays open but a 4+ subscription split is off the table. | Environment isolation comes from catalogs, bindings, and identities, not workspace count; extra workspaces multiply infrastructure and drift surface without adding isolation. | [Environments](databricks-environments.md) |
| D3 | 2026-08 | Single regional Unity Catalog metastore, both workspaces attached, no metastore-level root storage. A metastore-per-environment proposal was evaluated and rejected. | Platform limit, not preference: ["You can create only one metastore per region"](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore). | [Azure infrastructure](azure-infrastructure.md) |
| D4/D5 | 2026-08 | Repo naming conventions stand unchanged: suffix style for Azure resources, kind prefixes for identities, snake_case UC objects. Vendor-proposed names (`sp-ng-*`, `rg-rubicon-<env>-<workload>`, `sub-rubicon-*`) are rejected; names never encode the vendor or implementer. | Names outlive engagements; ownership is a tag (`owner`), not a name token. | [Naming conventions](naming-conventions.md) |
| D6 | 2026-08 | Storage layout: one ADLS container per medallion layer plus `landing` and `checkpoints` on the tier account, each layer container a Terraform-managed UC external location; table data at `<container>/<catalog>/<schema>/<table>`; `landing` and `checkpoints` back UC volumes. Replaces the undifferentiated managed-storage-per-catalog position. Layer placement is implemented through schema managed locations under the layer external locations, because pipeline-created streaming tables and materialized views are always managed tables (`LOCATION` is [not supported in pipeline definitions](https://learn.microsoft.com/en-us/azure/databricks/ldp/unity-catalog#limitations)). | Physical layout per layer makes storage governance, cost, and lifecycle legible per layer; the path convention keeps catalogs sharing the tier account on disjoint paths. | [Azure infrastructure](azure-infrastructure.md), [Medallion data practices](../practices/medallion-data-practices.md) |
| D7 | 2026-08 | Full classic-side private endpoint matrix adopted per tier (ADLS `blob`/`dfs`, Key Vault `vault`, Azure SQL `sqlServer`, Databricks `databricks_ui_api` and `browser_authentication`), with Private DNS zones centralized in the hub. NCC private endpoints continue to serve the serverless side. RDS and S3 are out of Azure Private Endpoint scope. | With D1's classic plane in place, the standard Azure Private Endpoint pattern is the sanctioned private path for it, in parallel with NCC for serverless. | [Azure infrastructure](azure-infrastructure.md) |

## Open items

Each needs an owner and a decision date; the owning doc carries the requirements. NG-driven items also appear in the [README outline](../../README.md); resolutions land here and in the owning doc by PR, never in a vendor document.

| Item | Question | Owner |
| --- | --- | --- |
| AWS-to-Azure connectivity model | VPN/ExpressRoute via the hub gateway, public endpoint with allowlist, cross-cloud PrivateLink, or landing data behind an Azure endpoint. Gates RDS/S3 onboarding. | NG (Dave Newman); counterpart Graham Lammers |
| Azure region | The `[region]` token, pending tenant review. | NG (Dave Newman); counterpart Graham Lammers |
| Azure SQL DB network location | Where the existing source lives and its reconciliation into the hub-and-spoke topology; decides its private endpoint placement. | NG (Dave Newman); counterpart Graham Lammers |
| IP address plan | Hub and spoke CIDRs validated against AWS VPC ranges and existing Rubicon networks; subnet CIDRs are immutable after deploy. | NG (Dave Newman); counterpart Graham Lammers |
| Subscription topology | (a) one shared subscription with tier resource groups plus hub resources, or (b) prod / nonprod / connectivity split. Constrained by D2; spoke VNets must share a subscription with their workspace. | TBD (Cloud team) |
| Storage redundancy / BCDR | Redundancy tiers, RTO/RPO, restore paths. Decision register: [Resilience](resilience.md). | TBD |
| Serverless egress policy | Account-level network policies to restrict serverless outbound, and the allowlist. | TBD |
| Azure Policy baseline | Required tags, allowed regions, public-network-access deny, diagnostic settings deployment. | TBD (Cloud team) |
| Log Analytics wiring | Diagnostic settings, retention, minimum alert set for `[subject]-dbx-log-*`. | TBD (DevOps team) |

## Recording rule

- A new decision gets a row here and its rules in the owning doc, in the same PR. A register row without normative content in a doc is incomplete; normative content without a register row is undiscoverable.
- A rejected alternative worth remembering is recorded in the decision's rationale, not as a separate row.
- When an open item closes, move it to the decisions table with its date; do not delete the history.

## Checklist

- [ ] Every decided row names its normative doc, and that doc carries the rules
- [ ] Every open item has an owner or an explicit TBD with the owning team
- [ ] No resolution exists only in a vendor document or meeting notes

## Sources

- Azure Databricks: [Create a Unity Catalog metastore](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore)
- Azure Databricks: [Use Unity Catalog with pipelines](https://learn.microsoft.com/en-us/azure/databricks/ldp/unity-catalog)
- Azure: [Azure Private Endpoint private DNS zone values](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)
