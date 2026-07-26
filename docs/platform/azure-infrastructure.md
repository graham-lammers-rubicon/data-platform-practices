# Azure Infrastructure

Defines the Azure footprint the platform runs on. All infrastructure is Terraform; a portal-created resource is a defect ([Environments](environments.md)). Names come from [Naming conventions](naming-conventions.md).

The platform standard is **serverless workspaces**: no customer-managed VNet, no classic compute plane, all workloads on serverless compute. [Generally available on Azure since January 2026](https://learn.microsoft.com/en-us/azure/databricks/admin/workspace/serverless-workspaces). This removes the network layer we would otherwise have to design, size, and operate (VNet injection, subnets, NAT, NSGs). A VNet-injected workspace is the fallback for the specific cases below, not the default.

## What this covers

- Resource layout per tier
- Serverless workspace model and its limits
- Reaching protected and on-premises resources from serverless compute
- Unity Catalog wiring and identity
- The Terraform standard

## Layout

Two tiers, one region, one resource group per tier.

| Resource | Nonprod | Prod |
| --- | --- | --- |
| Resource group | `rg-dbx-np-[region]-001` | `rg-dbx-prod-[region]-001` |
| Databricks workspace (serverless) | `dbw-dbx-np-[region]-001` | `dbw-dbx-prod-[region]-001` |
| Storage (ADLS Gen2, UC managed storage) | `stdbxnp[region]001` | `stdbxprod[region]001` |
| Access connector | `dbac-dbx-np-[region]-001` | `dbac-dbx-prod-[region]-001` |
| Platform key vault | `kv-dbx-np-[region]-001` | `kv-dbx-prod-[region]-001` |
| Log Analytics | `log-dbx-np-[region]-001` | `log-dbx-prod-[region]-001` |

Domain secret vaults (`kv-<domain>-<env>-[region]-001`) are created per secret scope as domains onboard ([Secrets and credentials](secrets-and-credentials.md)).

## Serverless workspaces

- Workspace type is Serverless: preconfigured with serverless compute and default storage, no VNet, no classic compute plane. Premium features (Unity Catalog, policies) apply as on any workspace.
- Data lives in Unity Catalog managed storage on the tier ADLS account, accessed through the access connector's managed identity. Default storage holds only workspace system data.
- Serverless workspaces run [environment versions](https://learn.microsoft.com/en-us/azure/databricks/admin/workspace/serverless-workspaces), not Databricks Runtime versions. Anything requiring classic compute (R, RDD APIs, JAR libraries, custom DBR) does not run here.
- **When to fall back to a VNet-injected workspace:** a workload that genuinely requires classic compute, or network control that serverless cannot express (customer-managed egress filtering, direct private connectivity to on-premises). Treat that as an architecture decision with an owner, not a convenience.

## Reaching protected and on-premises resources

Serverless compute runs in the Databricks tenant, so network access is configured through [network connectivity configurations (NCCs)](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/), account-level objects attached to the workspace. Terraform-managed like everything else.

- **Azure resources (storage, Key Vault, databases):** add a private endpoint to the NCC; approve the request on the target resource. Supported from SQL warehouses, jobs, pipelines, and model serving.
- **Locked-down storage without private endpoints:** allowlist the `AzureDatabricksServerless` service tag through an Azure Network Security Perimeter. The NSP requirement has been in force since 2026-06-09; do not build on serverless subnet-ID allowlists.
- **Resources inside a VNet:** NCC private endpoint to a Private Link service fronting an internal load balancer (or Application Gateway) in that VNet.
- **On-premises resources:** not a documented serverless connectivity target. Do not design a serverless workload that assumes a route to on-prem. The supported patterns, in preference order:
  1. Land the data behind an Azure endpoint serverless can reach: push or replicate from on-prem to ADLS, Event Hubs, or an Azure database, then ingest normally. This is the default answer.
  2. If compute must reach on-prem directly (pull-only source systems), run that ingestion on classic compute in a VNet-injected workspace with ExpressRoute/VPN, land the output in Bronze, and keep everything downstream serverless.

## Unity Catalog wiring

- One metastore per region is a [platform limit](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore). Both workspaces attach to it. Terraform manages it.
- No metastore-level root storage. Managed storage is declared per catalog, pointing at the tier storage account, so tier isolation holds at the storage layer.
- Storage access is via the tier access connector's managed identity registered as a UC storage credential. No service principal credentials for storage, nothing to rotate.
- The metastore admin role is held by a platform team group, not an individual.
- Catalog-workspace bindings enforce the tier boundary ([Environments](environments.md)); `prod_catalog` is `ISOLATED` before the first data lands.

## Identity

Identity is the platform's real security boundary; with serverless workspaces there is no network perimeter to lean on.

- Users and groups: Microsoft Entra ID via automatic identity management; grants attach to Entra-synced groups only ([Access model](../governance/access-model.md)).
- Automation: Databricks-managed service principals with OIDC federation, no stored credentials ([Service principal authentication](service-principal-auth.md)).
- Humans authenticate to the CLI with OAuth U2M profiles ([Service principal authentication](service-principal-auth.md#human-authentication)).
- Storage and Azure services: managed identities (access connector, Terraform deploy identity). No PATs, no client secrets where federation works.

## Terraform standard

- One infrastructure repo owns everything above plus identities, grants, policies, scopes, and workspace configuration. Providers: `azurerm` for Azure resources, `databricks` for account and workspace objects.
- The deploy identity is the Entra-federated managed identity from [Service principal authentication](service-principal-auth.md); plan and apply run in CI only.
- State is remote in an Azure storage account, encrypted, access-controlled, excluded from git. No secret values in configuration.
- Every resource carries the baseline tags (`env`, `domain`, `owner`, `costCenter`, `managedBy: terraform`).

## Sharp edges

- The first Databricks account admin must be a Microsoft Entra ID Global Administrator at first login to the account console. Plan that bootstrap step; it cannot be Terraformed.
- The Databricks-managed resource group is not modifiable and cannot host your resources.
- One metastore per region means a second region is a second metastore with a disjoint namespace. Region expansion is an architecture change, not a Terraform variable.
- Verify current `azurerm`/`databricks` provider support for the serverless workspace type before the first deploy; the offering is newer than most provider examples.
- NCC private-endpoint rules to Private Link services do not support DNS chasing; plan target domain names explicitly.
- The Network Security Perimeter requirement (in force since 2026-06-09) applies to any storage firewall still allowlisting serverless subnet IDs; migrate any legacy allowlist immediately.
- If a VNet-injected fallback workspace is ever built: subnet CIDRs are immutable after deployment and are a hard cluster-count ceiling, and new VNets need an explicit egress method (NAT gateway) since Azure's [default outbound retirement](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access) (API versions after 2026-03-31 default subnets to private). Size and design before deploying, not after.

## Checklist

- [ ] Every resource exists in Terraform state with baseline tags; portal diff is zero
- [ ] Workspaces are serverless type; any VNet-injected workspace has a documented owner and reason
- [ ] Every protected resource serverless reaches is wired through an NCC (private endpoint or NSP service tag), in Terraform
- [ ] No workload assumes direct serverless-to-on-prem connectivity
- [ ] Both workspaces attach to the single regional metastore; managed storage is per catalog
- [ ] Storage credentials use the tier access connector managed identity; zero SP-based storage access
- [ ] Metastore admin is a group
- [ ] Terraform state is remote, encrypted, and not in git

## Sources

- Azure Databricks: [Serverless workspaces](https://learn.microsoft.com/en-us/azure/databricks/admin/workspace/serverless-workspaces)
- Azure Databricks: [Serverless network security (NCCs)](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/)
- Azure Databricks: [Private connectivity from serverless compute](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/serverless-private-link)
- Azure Databricks: [Private Link to resources in your VNet](https://learn.microsoft.com/en-us/azure/databricks/security/network/serverless-network-security/pl-to-internal-network)
- Azure Databricks: [Create a Unity Catalog metastore](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore)
- Azure: [Default outbound access retirement](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)
