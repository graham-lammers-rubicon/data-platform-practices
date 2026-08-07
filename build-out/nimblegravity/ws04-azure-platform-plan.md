**NIMBLE GRAVITY**

# Rubicon Technologies Transformation Program

## Workstream 04: Data Foundation Build

### Azure Platform Plan (Draft, Placeholders Flagged)

| Document Control | |
|---|---|
| Program | Rubicon Technologies Transformation Program |
| Client | Rubicon Global, LLC |
| Workstream | 04: Data Foundation Build |
| Nimble Gravity Lead | Dave Newman |
| Document status | Draft for review, prepared August 6, 2026 |
| Companion documents | WS04 Architecture & Implementation Plan; WS04 BRD/Tech Spec |

---

## 0. Purpose and How to Read This

This document targets the platform build-out for Workstream 04 on both fronts: the Azure foundation (subscription/resource group topology, network design, per-environment resource inventory, service principals, private endpoint connectivity) and the Databricks workspace build-out that runs on it (workspace provisioning with VNet injection and serverless via NCC, Unity Catalog metastore and storage wiring, Access Connector identity, cluster policies, and the deployment identities behind Databricks Asset Bundles). It builds directly on the ingestion architecture and environment topology already agreed in the companion Architecture & Implementation Plan (separate Azure subscription and Databricks workspace per environment, single Azure tenant).

There are three decisions that require further clarification and are listed as placeholders in this document: the AWS-to-Azure connectivity model, the Azure region, and the network location of the existing Azure SQL DB source. Each is called out inline and again in Section 9. Everything else reflects concrete proposals in this plan.

---

## 1. Decisions & Recommendations Carried Into This Plan

| Decision | Status |
|---|---|
| Databricks compute network model | **Hybrid.** Each environment's workspace is VNet-injected (classic) to support workloads needing direct, controlled network paths, such as CDC/JDBC connections to source databases, while SQL warehouses and jobs that don't need custom networking run on serverless compute via a Network Connectivity Config (NCC) for private access to storage and Unity Catalog. |
| AWS-to-Azure connectivity for RDS/S3 sources | **Placeholder.** Depends on Rubicon's existing AWS networking posture, which is currently undocumented (this is the same Rackspace/Terraform knowledge gap flagged in the companion Architecture & Implementation Plan). See Section 4.4 and Section 9. |
| Network topology | **Hub-and-spoke with shared connectivity.** A hub VNet carries shared networking (firewall, gateway subnet, centralized Private DNS zones); dev, test, and prod each get a spoke VNet peered to the hub. |
| Azure region | **Placeholder**, pending Azure tenant review. Region should ideally sit near wherever RDS/S3 currently live, to control CDC replication latency and cross-cloud egress cost. All resource names/examples below use a placeholder region code (`<region>`). |

---

## 2. Subscription and Management Group Topology

### 2.1 Subscriptions

Four subscriptions, all under Rubicon's single Azure tenant:

| Subscription | Purpose |
|---|---|
| `sub-rubicon-connectivity` | Hosts the hub VNet, shared connectivity (VPN Gateway/ExpressRoute once the AWS model is confirmed), Azure Firewall, and centralized Private DNS zones. Isolated from workload subscriptions so shared-network changes don't ride on a workload change window. |
| `sub-rubicon-dev` | Dev environment: Databricks workspace, storage, supporting resources. |
| `sub-rubicon-test` | Test/QA environment, mirrors dev at a different scale. |
| `sub-rubicon-prod` | Production environment. |

**Recommendation, confirm at kickoff:** put the hub in its own subscription rather than folding it into `sub-rubicon-prod`. This keeps the isolation philosophy already proposed for dev/test/prod (Architecture & Implementation Plan, Section 4) consistent for shared networking too, and avoids coupling firewall/gateway changes to production's change control. The tradeoff is a fourth subscription to administer and a small amount of added peering/DNS-linking configuration.

### 2.2 Management groups

Given Rubicon's Azure footprint is immature and this is four subscriptions, not a large multi-team estate, a full Cloud Adoption Framework management group hierarchy is more governance than the workstream needs right now. Recommended minimum: a single custom management group (e.g. `mg-rubicon`) under Tenant Root, with all four subscriptions underneath it, so tenant-wide Azure Policy (allowed regions, required tags, private endpoint enforcement) can be assigned once. Expanding into a full landing zone hierarchy (platform/landing zones/sandbox) is a reasonable future hardening step, not a Build-phase requirement.

---

## 3. Resource Group Topology

Graham's "playbook" has slightly different naming standards. The recommendations below can be easily changed. Proposed naming convention: `rg-rubicon-<env>-<workload>` (region code omitted from RG names since each RG's resources share the same region; included at the resource level instead).

| Subscription | Resource Groups |
|---|---|
| `sub-rubicon-connectivity` | `rg-rubicon-hub-network` (hub VNet, gateway subnet, Azure Firewall), `rg-rubicon-hub-dns` (Private DNS zones) |
| `sub-rubicon-dev` / `-test` / `-prod` (same pattern in each) | `rg-rubicon-<env>-network` (spoke VNet, NSGs, route tables), `rg-rubicon-<env>-databricks` (Databricks workspace, Access Connector), `rg-rubicon-<env>-data` (ADLS Gen2, Unity Catalog storage), `rg-rubicon-<env>-security` (Key Vault), `rg-rubicon-<env>-monitoring` (Log Analytics workspace, diagnostic settings) |

---

## 4. Network Design

### 4.1 Hub VNet (`sub-rubicon-connectivity`)

| Subnet | Purpose |
|---|---|
| `GatewaySubnet` | Reserved for VPN Gateway or ExpressRoute Gateway. Deployed once the AWS connectivity model (Section 4.4) is confirmed; reserving the subnet now avoids re-addressing later. |
| `AzureFirewallSubnet` | Azure Firewall (or equivalent NVA), used as the forced-egress point for spoke VNets that don't have direct public egress. |
| `snet-hub-shared` | Any shared services (e.g. a DNS forwarder/resolver if Rubicon needs hybrid DNS resolution to on-prem/AWS-side DNS). |

Private DNS zones are hosted in `rg-rubicon-hub-dns` and linked to the hub VNet plus every spoke VNet, so all environments resolve private endpoint names consistently. Zones needed (Section 8 has the full mapping): `privatelink.blob.core.windows.net`, `privatelink.dfs.core.windows.net`, `privatelink.vaultcore.azure.net`, and the Databricks front-end zones (`privatelink.azuredatabricks.net` and the secure cluster connectivity relay zone) if Databricks Private Link is enabled per Section 8.

### 4.2 Spoke VNets (one per environment, peered to the hub)

| Subnet | Purpose |
|---|---|
| `snet-<env>-dbx-public` | Databricks VNet injection "public" delegated subnet (legacy naming; with Secure Cluster Connectivity enabled, no node in this subnet actually gets a public IP). Delegated to `Microsoft.Databricks/workspaces`. |
| `snet-<env>-dbx-private` | Databricks VNet injection "private" delegated subnet. Same delegation. |
| `snet-<env>-privatelink` | Hosts private endpoint NICs for storage, Key Vault, Azure SQL DB, and Databricks front-end Private Link (where enabled). |

Each spoke peers to the hub; a route table on the Databricks subnets sends non-VNet-local traffic to the hub firewall for controlled egress (artifact repositories, PyPI, Databricks control-plane endpoints, etc.). The exact FQDN/port rule set for Databricks control-plane and package-repository access changes periodically; validate the current list against Databricks' VNet injection documentation at implementation time rather than relying on a fixed list here.

### 4.3 Address space

Illustrative only, not yet validated against any existing Rubicon network or the AWS VPC CIDR hosting RDS/S3 (see Section 9, this is a hard blocker for finalizing the network):

| VNet | Placeholder CIDR |
|---|---|
| Hub | `10.100.0.0/16` |
| Dev spoke | `10.101.0.0/16` |
| Test spoke | `10.102.0.0/16` |
| Prod spoke | `10.103.0.0/16` |

### 4.4 AWS connectivity for RDS/S3 (placeholder)

Private endpoints are an Azure-side construct; they don't by themselves make the AWS leg of the connection private. Until Rubicon's existing AWS networking posture is known, three paths remain open, each with different implications for the hub design:

- **Site-to-site VPN or ExpressRoute/Direct Connect** between the AWS VPC and the hub VNet's `GatewaySubnet`. Fully private end to end; requires coordinated, non-overlapping IP ranges on both sides and AWS-side gateway configuration Rubicon may or may not already have.
- **Public endpoint + IP allowlist + TLS**, with RDS/S3 reachable over their public endpoints restricted to known egress IPs (the hub firewall's public IP, if using SNAT there) and encrypted in transit. Faster to stand up, but the AWS leg is not private, only the Azure-side resources are.
- **Cross-cloud PrivateLink interconnect**, the most complex and typically only justified if Rubicon already has other cross-cloud private connectivity needs.

This needs to be resolved before the hub's gateway subnet is actually built out, since it determines whether a VPN/ExpressRoute gateway gets deployed at all.

---

## 5. Azure Resource Inventory by Environment

Databricks Premium tier is required in every environment (Unity Catalog dependency). Where dev/test/prod differ meaningfully, both are listed; otherwise assume parity.

### 5.1 Compute and Databricks

| Resource | Purpose | Dev | Test | Prod |
|---|---|---|---|---|
| Azure Databricks Workspace | VNet-injected, Premium tier, one per environment, hybrid classic + serverless (NCC-enabled) | Standard config, smaller cluster policies | Mirrors prod config at smaller scale | Full config, autoscaling cluster policies |
| Databricks Access Connector | Managed identity granting Unity Catalog and job clusters access to the environment's ADLS Gen2 account without storage keys | 1 | 1 | 1 |
| Network Connectivity Config (NCC) | Enables private connectivity for serverless SQL warehouses/jobs to storage and Unity Catalog | 1, attached to workspace | 1 | 1 |
| Cluster policies | Enforce instance types, autotermination, spot usage | Permissive, cost-capped | Mirrors prod | Stricter, production SLA-oriented |

### 5.2 Storage and Unity Catalog

| Resource | Purpose | Dev | Test | Prod |
|---|---|---|---|---|
| ADLS Gen2 storage account (hierarchical namespace) | Unity Catalog metastore root storage plus medallion containers (`landing`, `bronze`, `silver`, `gold`) | Standard, LRS | Standard, LRS or ZRS | Standard, ZRS or GRS depending on Rubicon's recovery requirements (see Section 9). Nimble Gravity would recommend LRS in dev and test, and GRS in prod. There are significant cost implications to changing storage redudancy settings |
| Unity Catalog Metastore | Governance layer of record (see Section 6 for the per-environment vs. shared recommendation) | 1, environment-scoped | 1, environment-scoped | 1, environment-scoped |
| Storage containers | `landing` (raw pre-ingest), `bronze`, `silver`, `gold`, plus a `_checkpoints` container for streaming/SDP checkpoint state | Same structure across all environments | | |

### 5.3 Security and Secrets

| Resource | Purpose | Dev | Test | Prod |
|---|---|---|---|---|
| Azure Key Vault | Source connection secrets (JDBC credentials, API keys), certificates | Standard tier | Standard tier | Standard tier, tighter access policies and purge protection enabled |
| Private endpoints | See Section 8 | | | |
| NSGs | Attached to every subnet in the spoke VNet | Baseline rules | Baseline rules | Baseline rules plus stricter deny-by-default posture |

### 5.4 Governance and Monitoring

| Resource | Purpose | Dev | Test | Prod |
|---|---|---|---|---|
| Log Analytics Workspace | Databricks audit logs, NSG flow logs, storage diagnostics | 1 per environment | 1 per environment | 1 per environment |
| Diagnostic settings | Route Databricks workspace, storage, and Key Vault logs to the Log Analytics workspace | Enabled | Enabled | Enabled |
| Azure Policy assignments | Allowed regions, required tags, private endpoint enforcement (deny public network access on storage/Key Vault) | Applied at `mg-rubicon` | Applied at `mg-rubicon` | Applied at `mg-rubicon` |
| Cost Management budget | Subscription-level spend alerting | Lower threshold | Lower threshold | Higher threshold, matched to SOW estimate |

---

## 6. Unity Catalog and Storage Layout

**Recommendation, confirm at kickoff:** one Unity Catalog metastore per environment, each bound only to that environment's single workspace. This mirrors the full-isolation philosophy already designed for subscriptions and workspaces (Architecture & Implementation Plan, Section 4). The alternative, a single shared metastore across all three workspaces with isolation enforced only through catalog-level permissions, is technically supported and reduces metastore administration overhead, but reintroduces a shared security boundary that the subscription/workspace split was specifically proposed to avoid.

Each environment's ADLS Gen2 account backs its own metastore as the default managed storage location, with additional external locations registered per medallion layer if the build ends up needing external (unmanaged) tables alongside Unity Catalog-managed tables. The `_checkpoints` container keeps Spark Declarative Pipeline / streaming checkpoint state out of the data containers.

---

## 7. Service Principals and Identities Required

Recommended pattern: use **Microsoft Entra Workload Identity Federation (OIDC)** between GitHub Actions and each Azure AD app registration, rather than long-lived client secrets stored in GitHub. This removes secret rotation as an operational burden and matches the production-grade bar for CI/CD credentials.

| Identity | Purpose | Scope | Notes |
|---|---|---|---|
| `sp-ng-terraform-<env>` | Provisions Azure infrastructure (networking, storage, Key Vault, Databricks workspace resource) via Terraform | Contributor on the environment's resource groups; User Access Administrator scoped narrowly where it must assign roles (e.g. Access Connector to storage) | One per environment; avoid a single subscription-wide or tenant-wide Contributor principal |
| `sp-ng-dab-deploy-<env>` | Deploys jobs, pipelines, and workflow definitions via Databricks Asset Bundles from CI/CD | Databricks workspace-level permissions (not a full workspace admin); Unity Catalog grants scoped to the catalogs/schemas the pipelines touch | One per environment |
| `sp-ng-cicd-github` (or per-environment equivalents) | GitHub Actions federated identity for triggering deployments | Federated credential trust to the specific GitHub repo/branch/environment | Prefer environment-scoped federated credentials (one app registration, multiple federated credentials) over one broad credential covering all branches |
| `sp-ng-powerbi-<env>` | Power BI service connection to Databricks SQL warehouses for the gold layer | Databricks SQL warehouse "Can Use" permission; Unity Catalog SELECT on gold catalog only | Likely only needed for test and prod; confirm whether dev needs BI connectivity at all. This strategy may change depending on OneLake Mirroring implementation |
| Databricks Access Connector (managed identity, not an app registration) | Unity Catalog and cluster access to ADLS Gen2 without storage keys | Storage Blob Data Contributor on the environment's storage account | One per environment; already listed in Section 5.2, repeated here because it's the primary storage-access identity, not the service principals above |

**Open question folded in here:** whether to split Terraform and Databricks Asset Bundle deployment into separate service principals per environment (as above, more granular least-privilege, more identities to manage) or consolidate into one deploy identity per environment covering both IaC and DAB. The split is the safer default for a production-grade bar; flagging in case Rubicon's smaller Azure footprint makes the simpler, consolidated option preferable during Build before tightening later.

---

## 8. Private Endpoint Connectivity Requirements

| Resource | Sub-resource / Endpoint Type | Environments | Private DNS Zone | Notes |
|---|---|---|---|---|
| ADLS Gen2 storage account | `blob` | dev, test, prod | `privatelink.blob.core.windows.net` | |
| ADLS Gen2 storage account | `dfs` | dev, test, prod | `privatelink.dfs.core.windows.net` | Required in addition to `blob` for Databricks/Unity Catalog hierarchical namespace access |
| Azure Key Vault | `vault` | dev, test, prod | `privatelink.vaultcore.azure.net` | |
| Azure SQL Database (existing source) | `sqlServer` | Whichever environment(s) need direct access | `privatelink.database.windows.net` | Contingent on Section 9's open item: which subscription/VNet currently hosts this resource, and whether it needs to move or simply be peered to |
| Azure Databricks Workspace (front-end/API) | `databricks_ui_api` | dev, test, prod (recommend prod first if sequencing is needed) | `privatelink.azuredatabricks.net` | Optional relative to VNet injection alone; in scope because private endpoints are a stated project requirement, not just data-plane isolation |
| Azure Databricks Workspace (secure cluster connectivity relay) | `browser_authentication` | dev, test, prod | Databricks-managed relay zone, confirm current zone name against Databricks docs at implementation time | Pairs with the front-end private endpoint above |
| Log Analytics Workspace (optional but recommended) | Azure Monitor Private Link Scope (AMPLS) | prod (recommended minimum); dev/test optional | Azure Monitor private link zones | Not required for baseline private endpoint coverage; flagged as a hardening option since monitoring data otherwise transits the Azure backbone rather than the public internet regardless |

**Explicitly out of scope for Azure Private Endpoint:** Amazon RDS and Amazon S3. Private Endpoint is an Azure construct that privatizes access to an Azure PaaS resource; it has no direct equivalent for AWS-hosted sources. Whatever privacy the RDS/S3 connection gets comes from the AWS connectivity model in Section 4.4, not from anything in this table.

---

## 9. Open Items Requiring Confirmation

| Item | Rationale | Owner |
|---|---|---|
| AWS connectivity model for RDS/S3 | Determines whether the hub's `GatewaySubnet` gets an actual gateway deployed, and whether a VPN/ExpressRoute cost line exists at all | Rubicon (depends on existing AWS networking posture, tied to the Rackspace/Terraform handoff gap) |
| Azure region | Affects latency to AWS sources, egress cost if a cross-cloud path is built, and every resource's actual location | Rubicon, pending Azure tenant review |
| Existing Azure SQL DB network location | Unknown which subscription/VNet currently hosts this source; determines whether it needs peering into the new hub-and-spoke topology or a migration | Rubicon |
| IP address plan | Placeholder CIDRs in Section 4.3 must be checked against the AWS VPC ranges and any other existing Rubicon networks before anything is deployed | Rubicon, with Nimble Gravity |
| Hub subscription vs. hub-in-prod | Recommended as a separate `sub-rubicon-connectivity` subscription (Section 2.1); confirm Rubicon is comfortable with a fourth subscription | Dave / Rubicon |
| Unity Catalog metastore-per-environment | Recommended (Section 6); confirm before provisioning, since consolidating later is more disruptive than starting consolidated | Dave / Rubicon |
| Storage redundancy for prod (LRS/ZRS/GRS) | Depends on Rubicon's recovery point/time objectives, not yet defined | Rubicon |
| Service principal consolidation (Section 7) | Split vs. combined Terraform/DAB identities per environment | Dave |
| Databricks Private Link (front-end) sequencing | Whether to enable on all three environments at once or prod-first | Dave / Rubicon |
