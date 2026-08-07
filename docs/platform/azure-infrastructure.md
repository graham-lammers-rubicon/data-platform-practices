# Azure Infrastructure

Defines the Azure footprint the platform runs on. All infrastructure is Terraform; a portal-created resource is a defect ([Environments](databricks-environments.md)). Names come from [Naming conventions](naming-conventions.md).

The compute model is **hybrid** (decided 2026-08, [Decision register](decision-register.md)): each workspace is VNet-injected, so the classic compute plane is available for workloads that need a direct, controlled network path, primarily CDC/JDBC pull connections to source databases (Azure SQL Database, Amazon RDS). Serverless compute, enabled through network connectivity configurations (NCCs), remains the preferred default for SQL warehouses, jobs, and pipelines that do not need custom networking. There is no separate fallback workspace; the standard workspaces carry both planes. Choosing classic compute for a workload still requires a stated reason in its spec.

The platform deliberately leans into newer Databricks capabilities (usage policies, metric views, Lakebase branching). Each preview or beta dependency names its fallback where it is used; the bet is early adoption with a stated exit, not early adoption on faith.

## What this covers

- Resource layout per tier
- Network design: hub and spoke
- The hybrid workspace model: VNet injection plus serverless via NCC
- Private endpoints and DNS for the classic side
- Reaching protected and remote resources from serverless compute
- Unity Catalog wiring and the storage layout
- Identity and the Terraform standard

## Layout

Two tiers, one region, one resource group per tier, plus shared connectivity resources in the hub.

| Resource | Nonprod | Prod |
| --- | --- | --- |
| Resource group | `[subject]-dbx-rg-[region]-nonprod` | `[subject]-dbx-rg-[region]-prod` |
| Databricks workspace (VNet-injected, serverless via NCC) | `[subject]-dbx-workspace-[region]-nonprod` | `[subject]-dbx-workspace-[region]-prod` |
| Spoke virtual network | `[subject]-dbx-vnet-[region]-nonprod` | `[subject]-dbx-vnet-[region]-prod` |
| Storage (ADLS Gen2, tier data) | `[subject]dbxstore[region]np` | `[subject]dbxstore[region]prod` |
| Access connector | `[subject]-dbx-connector-[region]-nonprod` | `[subject]-dbx-connector-[region]-prod` |
| Platform key vault | `[subject]-dbx-vault-[region]-np` | `[subject]-dbx-vault-[region]-prod` |
| Log Analytics | `[subject]-dbx-log-[region]-nonprod` | `[subject]-dbx-log-[region]-prod` |

Names follow the suffix pattern `[subject]-[scope]-<type>-[region]-<env>`: no instance counters, environment last ([Naming conventions](naming-conventions.md)). Shared connectivity resources (hub) use `shared` as the final token: `[subject]-hub-vnet-[region]-shared`. Domain secret vaults (`[subject]-<domain>-vault-[region]-<env>`) are created per secret scope as domains onboard ([Secrets and credentials](secrets-and-credentials.md)).

## Management group

All platform subscriptions sit under a single custom management group under Tenant Root, `[org]-mg` ([Naming conventions](naming-conventions.md)); however the open subscription topology decision lands, its subscriptions go under this group. The Azure Policy baseline (which policies is still open, below) is assigned once at the management group, never per subscription, so guardrails cannot drift between subscriptions. A full Cloud Adoption Framework landing-zone hierarchy is deferred: it is more governance than an estate of this size needs, and adopting one later is additive, not rework. Adopted 2026-08 from the vendor Azure Platform Plan, section 2.2 ([Decision register](decision-register.md)).

## Network design: hub and spoke

One hub VNet plus one spoke VNet per workspace tier, owned jointly with the Cloud team.

- **Hub** (`[subject]-hub-vnet-[region]-shared`): Azure Firewall or an equivalent NVA as the forced-egress point for the spokes; the `GatewaySubnet` reserved for the AWS connectivity gateway (VPN or ExpressRoute; the AWS connectivity model itself is an open decision, see [Decision register](decision-register.md)); the centralized Private DNS zones (next section).
- **Spokes** (`[subject]-dbx-vnet-[region]-nonprod`, `[subject]-dbx-vnet-[region]-prod`): peered to the hub, one per workspace tier. Route tables (`[subject]-dbx-rt-[region]-<env>`) on the Databricks subnets send non-local traffic to the hub firewall.
- **Spoke subnets per tier:** two delegated Databricks subnets (host/public and container/private, delegation `Microsoft.Databricks/workspaces`, secure cluster connectivity enabled) and one private-endpoint subnet. Databricks [requires the two dedicated subnets](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject) and recommends nothing smaller than `/26` for each; the VNet CIDR must be between `/16` and `/24`.
- **CIDRs are placeholders** until the IP address plan is validated against the AWS VPC ranges and existing networks (open item, [Decision register](decision-register.md)). Subnet CIDRs are immutable after deployment; size before deploying, not after.
- The spoke VNet must be in the same region and the same subscription as its workspace ([VNet injection requirements](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)). This constrains the subscription topology decision below.

## Hybrid workspaces

- Workspace type is VNet-injected with secure cluster connectivity. Serverless compute is enabled on the same workspace and reaches protected resources through NCCs. Both planes are standard; neither is a fallback.
- Serverless is the preferred compute for every workload class ([Compute policies](databricks-compute-policies.md)). Classic compute exists for workloads that need the classic network path: CDC/JDBC pull ingestion from source databases over private connectivity, or capabilities serverless does not support. The reason is recorded in the workload's spec.
- Classic compute egress is forced through the hub firewall via the subnet route tables. The firewall must carry the [user-defined route allowances Databricks documents](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/udr) for control-plane traffic; a missing allowance breaks cluster launch.
- Data lives on the tier ADLS account, governed through Unity Catalog external locations and managed locations (storage layout below), accessed through the access connector's managed identity.

## Private endpoints and DNS (classic side)

The classic compute plane and users reach protected resources through standard Azure Private Endpoints, deployed per tier into the spoke's private-endpoint subnet. The serverless side reaches the same resources through NCC private endpoint rules (next section); the two are parallel paths to the same firewall-locked resource.

| Resource | Sub-resource | Private DNS zone |
| --- | --- | --- |
| ADLS Gen2 (tier storage) | `blob` | `privatelink.blob.core.windows.net` |
| ADLS Gen2 (tier storage) | `dfs` | `privatelink.dfs.core.windows.net` |
| Key Vault | `vault` | `privatelink.vaultcore.azure.net` |
| Azure SQL Database (source) | `sqlServer` | `privatelink.database.windows.net` |
| Databricks workspace front end | `databricks_ui_api` | `privatelink.azuredatabricks.net` |
| Databricks browser auth relay | `browser_authentication` | `privatelink.azuredatabricks.net` |

Sub-resource names and zones verified against the [Azure private endpoint DNS reference](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) and [Azure Databricks Private Link concepts](https://learn.microsoft.com/en-us/azure/databricks/security/network/concepts/private-link).

- Private DNS zones are centralized in the hub and linked to the hub and both spokes. One zone per service for the platform, not one per tier.
- The Azure SQL Database endpoint is contingent on where that source currently lives; its network location and reconciliation into this topology is an open item ([Decision register](decision-register.md)).
- Amazon RDS and Amazon S3 are explicitly out of scope for Azure Private Endpoint. Their privacy comes from the AWS connectivity model, which remains an open decision.

## Reaching protected and remote resources from serverless

Serverless compute runs in the Databricks tenant, so its network access is configured through [network connectivity configurations (NCCs)](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/), account-level objects attached to the workspace. Terraform-managed like everything else.

- **Azure resources (storage, Key Vault, databases):** add a private endpoint to the NCC; approve the request on the target resource. Supported from SQL warehouses, jobs, pipelines, and model serving.
- **Locked-down storage without private endpoints:** allowlist the `AzureDatabricksServerless` service tag through an Azure Network Security Perimeter. The NSP requirement has been in force since 2026-06-09; do not build on serverless subnet-ID allowlists.
- **Resources inside a VNet:** NCC private endpoint to a Private Link service fronting an internal load balancer (or Application Gateway) in that VNet.
- **Remote sources (Amazon RDS, S3, on-premises):** not a documented serverless connectivity target. Do not design a serverless workload that assumes a route to them. The supported patterns, in preference order:
  1. Land the data behind an Azure endpoint serverless can reach: push or replicate to ADLS, Event Hubs, or an Azure database, then ingest normally.
  2. If compute must reach the source directly (pull-only source systems), run that ingestion on classic compute in the standard workspace, over the hub's gateway connectivity, land the output in Bronze, and keep everything downstream serverless.

  Which pattern applies to RDS and S3 depends on the open AWS connectivity decision ([Decision register](decision-register.md)); do not build source onboarding that presumes one before it lands.

## Unity Catalog wiring

- One metastore per region is a platform limit: ["You can create only one metastore per region"](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore). Both workspaces attach to the single regional metastore. Terraform manages it. A metastore-per-environment design was evaluated and superseded 2026-08 for this reason; it is not a choice available within one region.
- No metastore-level root storage. Managed storage is declared per catalog and per layer schema on the tier storage account (layout below), so tier isolation holds at the storage layer.
- Storage access is via the tier access connector's managed identity registered as a UC storage credential. No service principal credentials for storage, nothing to rotate.
- The metastore admin role is held by a platform team group, not an individual.
- Catalog-workspace bindings enforce the tier boundary ([Environments](databricks-environments.md)); `prod_catalog` is `ISOLATED` before the first data lands.

## Storage layout

The tier ADLS account carries one container per medallion layer plus supporting containers (decided 2026-08, [Decision register](decision-register.md)):

| Container | Registered as | Holds |
| --- | --- | --- |
| `landing` | UC external location backing the `landing` volume | Source file drops awaiting ingestion |
| `bronze` | UC external location | Bronze table data |
| `silver` | UC external location | Silver table data |
| `gold` | UC external location | Gold table data |
| `checkpoints` | UC external location backing the `checkpoints` volume | Streaming checkpoints |

Rules:

- Every external location is backed by the tier access connector's storage credential and is Terraform-managed, as are the `landing` and `checkpoints` volumes.
- **Layer placement works through managed locations, not external tables.** Streaming tables and materialized views created by Lakeflow SDP pipelines are always managed tables; [the `LOCATION` property is not supported in pipeline definitions, and their data is stored in the containing schema's storage location](https://learn.microsoft.com/en-us/azure/databricks/ldp/unity-catalog#limitations). Each layer schema therefore declares its managed location under its layer's external location, and table data lands at `<container>/<catalog>/<schema>/<table>`.
- Catalogs sharing the tier storage account never share paths: the `<container>/<catalog>/<schema>` managed-location convention guarantees disjoint paths per catalog per layer.
- A table created outside a pipeline (for example a Gold serving table built by a job) MAY be an external table at an explicit path following the same convention, when a stated need exists (external reader access, existing data). Otherwise create managed tables.
- External tables [do not get predictive optimization](https://learn.microsoft.com/en-us/azure/databricks/optimizations/predictive-optimization): schedule `OPTIMIZE`, `VACUUM`, and `ANALYZE` for them in the domain maintenance job (`<domain>-maintenance`, [Naming conventions](naming-conventions.md)). Managed tables keep predictive optimization; this is a standing reason to prefer them.
- `DROP TABLE` on an external table [removes metadata but leaves the data files at the path](https://learn.microsoft.com/en-us/azure/databricks/tables/external). Deletion and erasure procedures must delete the path as an explicit step ([Data lifecycle](../governance/data-lifecycle.md), decision open).
- Direct path access remains forbidden for consumers. UC external location and volume grants are the only sanctioned path access; a consumer holding a storage account key or SAS token is a defect.

## Identity

Identity is the platform's primary security boundary. The network perimeter protects paths; UC grants decide who reads data.

- Users and groups: Microsoft Entra ID via automatic identity management; grants attach to Entra-synced groups only ([Access model](../governance/access-model.md)).
- Automation: Databricks-managed service principals with OIDC federation, no stored credentials ([Service principal authentication](databricks-service-principal-auth.md)).
- Humans authenticate to the CLI with OAuth U2M profiles ([Service principal authentication](databricks-service-principal-auth.md#human-authentication)).
- Storage and Azure services: managed identities (access connector, Terraform deploy identity). No PATs, no client secrets where federation works.

## Terraform standard

- One infrastructure repo owns everything above plus identities, grants, policies, scopes, and workspace configuration. Providers: `azurerm` for Azure resources, `databricks` for account and workspace objects.
- The deploy identity is the Entra-federated managed identity from [Service principal authentication](databricks-service-principal-auth.md); plan and apply run in CI only.
- State is remote in an Azure storage account, encrypted, access-controlled, excluded from git. No secret values in configuration.
- Every resource carries the baseline tags (`env`, `domain`, `owner`, `costCenter`, `managedBy: terraform`).

## Open decisions (with the Cloud and DevOps teams)

These decisions are open, owned jointly with the Cloud and DevOps teams; each needs an owner and a decision date. Requirements stated, answers not. The consolidated open list, including the vendor-driven items, lives in the [Decision register](decision-register.md).

| Decision | Question | Owner |
| --- | --- | --- |
| AWS-to-Azure connectivity | How classic compute reaches Amazon RDS and S3: VPN or ExpressRoute via the hub gateway, public endpoint with allowlist, or landing the data behind an Azure endpoint. Gates source onboarding design. | TBD |
| IP address plan | Hub and spoke CIDRs validated against AWS VPC ranges and existing networks. Subnet CIDRs are immutable; this lands before the first VNet deploys. | TBD |
| Azure region | The `[region]` token, pending tenant review. | TBD |
| Azure SQL DB source location | Where the existing Azure SQL DB source lives and how it reconciles into the hub-and-spoke topology; decides its private endpoint placement. | TBD |
| Serverless egress control | Serverless compute has unrestricted outbound by default. Do we adopt Databricks account-level network policies to restrict egress, and to what allowlist? | TBD |
| Subscription topology | Two candidates remain: (a) one shared subscription with tier resource groups plus the connectivity hub resources, or (b) a prod / nonprod / connectivity subscription split. A subscription-per-environment design (4+ subscriptions) is off the table (decided 2026-08, [Decision register](decision-register.md)). Note: each spoke VNet must live in the same subscription as its workspace. | TBD |
| Azure Policy baseline | Which policies are assigned (required tags, allowed regions, public-network-access deny, diagnostic settings deployment)? Assignment scope is decided: once, at the management group (see above). | TBD |
| Log Analytics wiring | Which diagnostic settings feed the `[subject]-dbx-log-*` workspaces, what retention, what minimum alert set? Today the workspace is named but unused. | TBD |
| BCDR | Redundancy tiers and recovery: decision register in [Resilience](resilience.md). | TBD |

## Sharp edges

- Subnet CIDRs are immutable after deployment and are a hard cluster-count ceiling. New VNets need an explicit egress method since Azure's [default outbound retirement](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access) (API versions after 2026-03-31 default subnets to private); here the hub firewall route is that method, and it must exist before the first cluster launches. These are load-bearing constraints of the standard workspaces, not fallback notes.
- Forcing egress through the firewall without the documented [UDR allowances](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/udr) for the Databricks control plane silently breaks cluster launch and metastore access.
- Only one `browser_authentication` private endpoint can exist per region and private DNS zone, and [deleting its host workspace breaks web login for every workspace sharing that zone](https://learn.microsoft.com/en-us/azure/databricks/security/network/concepts/private-link). Decide its host deliberately and record it.
- The first Databricks account admin must be a Microsoft Entra ID Global Administrator at first login to the account console. Plan that bootstrap step; it cannot be Terraformed.
- The Databricks-managed resource group is not modifiable and cannot host your resources; the delegated subnets cannot host anything but the workspace.
- One metastore per region means a second region is a second metastore with a disjoint namespace. Region expansion is an architecture change, not a Terraform variable.
- NCC private-endpoint rules to Private Link services do not support DNS chasing; plan target domain names explicitly.
- The Network Security Perimeter requirement (in force since 2026-06-09) applies to any storage firewall still allowlisting serverless subnet IDs; migrate any legacy allowlist immediately.
- A pipeline table cannot be created as an external table; designs that assume path-addressed Bronze/Silver/Gold tables (rather than schema managed locations) will fail at the first `CREATE FLOW`. Route layout requirements through managed locations.

## Checklist

- [ ] Every resource exists in Terraform state with baseline tags; portal diff is zero
- [ ] All platform subscriptions sit under `[org]-mg`; the Azure Policy baseline is assigned at the management group, not per subscription
- [ ] Workspaces are VNet-injected with secure cluster connectivity and serverless enabled via NCC; no separate fallback workspace exists
- [ ] Hub and spoke VNets peered; Databricks subnets delegated, routed to the hub firewall, with the documented UDR allowances in place
- [ ] Classic-side private endpoints deployed per the matrix; Private DNS zones centralized in the hub and linked to hub and spokes
- [ ] Every protected resource serverless reaches is wired through an NCC (private endpoint or NSP service tag), in Terraform
- [ ] Any workload on classic compute has the reason recorded in its spec
- [ ] No workload assumes serverless connectivity to RDS, S3, or on-premises sources
- [ ] Both workspaces attach to the single regional metastore; layer schemas declare managed locations under their layer external locations
- [ ] All five containers exist per tier; external locations and volumes are Terraform-managed; no consumer holds path credentials
- [ ] External tables (if any) are covered by the domain maintenance job
- [ ] Storage credentials use the tier access connector managed identity; zero SP-based storage access
- [ ] Metastore admin is a group
- [ ] Terraform state is remote, encrypted, and not in git

## Sources

- Azure Databricks: [Deploy Azure Databricks in your Azure virtual network (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)
- Azure Databricks: [User-defined route settings](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/udr)
- Azure Databricks: [Azure Private Link concepts](https://learn.microsoft.com/en-us/azure/databricks/security/network/concepts/private-link)
- Azure: [Azure Private Endpoint private DNS zone values](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)
- Azure Architecture Center: [Hub-spoke network topology in Azure](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke)
- Azure: [What are Azure management groups?](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- Azure Databricks: [Serverless network security (NCCs)](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/)
- Azure Databricks: [Private connectivity from serverless compute](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/serverless-private-link)
- Azure Databricks: [Private Link to resources in your VNet](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/pl-to-internal-network)
- Azure Databricks: [Create a Unity Catalog metastore](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore)
- Azure Databricks: [Use Unity Catalog with pipelines](https://learn.microsoft.com/en-us/azure/databricks/ldp/unity-catalog)
- Azure Databricks: [Connect to an ADLS Gen2 external location](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/external-locations-adls)
- Azure Databricks: [Work with external tables](https://learn.microsoft.com/en-us/azure/databricks/tables/external)
- Azure Databricks: [Predictive optimization for Unity Catalog managed tables](https://learn.microsoft.com/en-us/azure/databricks/optimizations/predictive-optimization)
- Azure: [Default outbound access retirement](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)
