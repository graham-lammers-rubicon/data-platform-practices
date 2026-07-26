# Azure Infrastructure

Defines the Azure footprint the platform runs on: what exists, how it is laid out, and the rules that keep it reproducible. All infrastructure is Terraform; a portal-created resource is a defect ([Environments](environments.md)). Names come from [Naming conventions](naming-conventions.md).

## What this covers

- Resource layout per tier
- Workspace provisioning: VNet injection and secure cluster connectivity
- Unity Catalog wiring: metastore, access connectors, storage
- The Terraform standard

## Layout

Two tiers, one region, one resource group per tier plus the Databricks-managed groups Azure creates. `[region]` is a placeholder throughout; the real region token is set in Terraform ([Naming conventions](naming-conventions.md)).

| Resource | Nonprod | Prod |
| --- | --- | --- |
| Resource group | `rg-dbx-np-[region]-001` | `rg-dbx-prod-[region]-001` |
| Databricks workspace | `dbw-dbx-np-[region]-001` | `dbw-dbx-prod-[region]-001` |
| Virtual network | `vnet-dbx-np-[region]-001` | `vnet-dbx-prod-[region]-001` |
| Storage (ADLS Gen2) | `stdbxnp[region]001` | `stdbxprod[region]001` |
| Access connector | `dbac-dbx-np-[region]-001` | `dbac-dbx-prod-[region]-001` |
| Platform key vault | `kv-dbx-np-[region]-001` | `kv-dbx-prod-[region]-001` |
| Log Analytics | `log-dbx-np-[region]-001` | `log-dbx-prod-[region]-001` |

Domain secret vaults (`kv-<domain>-<env>-[region]-001`) are created per secret scope as domains onboard ([Secrets and credentials](secrets-and-credentials.md)).

## Workspace provisioning

- Workspaces deploy with VNet injection into the tier VNet: a host subnet and a container subnet, delegated to `Microsoft.Databricks/workspaces`, each at least `/26`. Databricks [does not recommend smaller](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject), and subnets cannot be shared across workspaces.
- Secure cluster connectivity is enabled: no public IPs on cluster nodes; both subnets use private IPs.
- Egress goes through an Azure NAT gateway on both subnets for stable egress IPs. This is [required for new VNets after 2026-03-31](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject), when Azure retires default outbound access.
- NSGs are auto-provisioned through subnet delegation; the documented rules are not modified. One NSG per workspace.
- Premium plan: Unity Catalog attachment, compute policies, and Creator-scoped secret scopes all require it.

## Unity Catalog wiring

- One metastore per region is a platform limit, not a choice: ["You can create only one metastore per region"](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore). Both workspaces attach to the single regional metastore. Terraform manages it (`databricks_metastore`, `databricks_metastore_assignment`).
- No metastore-level root storage. Managed storage is declared per catalog, pointing at the tier storage account (`uc-managed` container), so tier isolation holds at the storage layer too.
- Storage access is via the tier access connector's managed identity, registered as a UC storage credential. Databricks [strongly recommends managed identities](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore) over service principals for storage access: no credential to rotate, works behind a storage firewall.
- The metastore admin role is held by a platform team group, not an individual, per the [Databricks recommendation](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore).
- Catalog-workspace bindings enforce the tier boundary ([Environments](environments.md)); `prod_catalog` is `ISOLATED` before the first data lands.

## Terraform standard

- One infrastructure repo owns everything above plus identities, grants, policies, scopes, and workspace configuration ([Environments](environments.md), everything-as-code).
- Providers: `azurerm` for Azure resources, `databricks` for account and workspace objects. The deploy identity is the Entra-federated managed identity from [Service principal authentication](service-principal-auth.md); plan and apply run in CI only.
- State is remote in an Azure storage account, encrypted, access-controlled, and excluded from git, per [HashiCorp guidance](https://developer.hashicorp.com/terraform/language/state/sensitive-data). No secret values in configuration ([Secrets and credentials](secrets-and-credentials.md)).
- Every resource carries the baseline tags (`env`, `domain`, `owner`, `costCenter`, `managedBy: terraform`).

## Sharp edges

- Subnet CIDR ranges cannot be changed after workspace deployment, and each cluster node consumes two IPs (one per subnet) with five Azure-reserved IPs per subnet. An undersized subnet is a hard cluster-count ceiling fixed only by a Public Preview network update or a support request. Size for the ceiling, not the average.
- The first Databricks account admin must be a Microsoft Entra ID Global Administrator at first login to the account console. Plan that bootstrap step; it cannot be Terraformed.
- The Databricks-managed resource group is not modifiable and cannot host your resources. Everything you own goes in the tier resource group.
- One metastore per region means a second region is a second metastore with a disjoint namespace; cross-region sharing goes through open sharing, not a shared catalog. Region expansion is an architecture change, not a Terraform variable.
- The NAT gateway is load-bearing after the Azure default-outbound retirement: a new VNet without an explicit egress method produces clusters that cannot reach the control plane.
- GitHub-hosted runners are outside these VNets. Anything they must reach (state storage, vaults) needs firewall rules or self-hosted runners; see the reachability edge in [Secrets and credentials](secrets-and-credentials.md).

## Checklist

- [ ] Every resource exists in Terraform state with baseline tags; portal diff is zero
- [ ] Workspaces are VNet-injected with secure cluster connectivity and NAT gateway egress
- [ ] Subnets are `/26` or larger, delegated, one workspace pair per subnet set
- [ ] Both workspaces attach to the single regional metastore; managed storage is per catalog
- [ ] Storage credentials use the tier access connector managed identity; zero SP-based storage access
- [ ] Metastore admin is a group
- [ ] Terraform state is remote, encrypted, and not in git

## Sources

- Azure Databricks: [Deploy Azure Databricks in your Azure virtual network (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)
- Azure Databricks: [Create a Unity Catalog metastore](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-metastore)
- Azure Databricks: [Create service credentials](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-services/service-credentials) (access connector pattern)
- HashiCorp: [Sensitive data in Terraform state](https://developer.hashicorp.com/terraform/language/state/sensitive-data)
