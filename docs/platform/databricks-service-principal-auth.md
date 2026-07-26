# Service Principal Authentication

Defines the service principal model: which type to use, how each identity authenticates, and its lifecycle. Human identities never run production workloads. Provisioning policy for users and groups (automatic identity management) lives in the [Access model](../governance/access-model.md); this doc covers the automation identities.

## What this covers

- The two service principal types and when each applies
- The platform's standard identities and their auth paths
- Auth method ranking and credential rules
- Lifecycle: creation, deactivation, deletion

## Service principal types

Azure Databricks has two kinds of service principal:

- **Databricks-managed**: created and managed inside Databricks. Authenticates with Databricks OAuth, including workload identity federation.
- **Microsoft Entra ID managed**: an Entra app registration linked into Databricks by its application (client) ID. Authenticates with Databricks OAuth or Entra tokens.

The [Databricks recommendation](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals), adopted here as a rule: use Databricks-managed service principals for Databricks automation; use Entra-managed service principals only "in cases where you must authenticate with Azure Databricks and other Azure resources at the same time."

## Standard identities

| Identity | Type | Auth | Purpose |
| --- | --- | --- | --- |
| `sp-dbx-deploy-np`, `sp-dbx-deploy-prod` | Databricks-managed | GitHub OIDC via Databricks federation policy | Bundle deploys ([CI/CD](github-cicd-and-deployment.md)) |
| `sp-<domain>-pipeline-<env>` | Databricks-managed | None held; runs as `run_as` in the bundle | Pipeline and job execution |
| `id-dbx-deploy-<tier>-001` | Entra user-assigned managed identity | GitHub OIDC via Entra federated credential | Terraform: ARM plus Databricks account provisioning |
| `sp-<app>-<env>` | Databricks-managed | OAuth M2M | Apps and external callers reading Gold |

Name patterns: [Naming conventions](naming-conventions.md).

## Human authentication

Humans authenticate the Databricks CLI and IDE integrations with OAuth user-to-machine (U2M), never PATs.

- `databricks auth login --host https://<workspace-url> --profile <name>` runs a browser login and writes a profile to `~/.databrickscfg` with auth type `databricks-cli`. Tokens are short-lived (under an hour) and refreshed by the CLI.
- From CLI v1.0, tokens live in OS-native secure storage; `.databrickscfg` holds only host and profile name, never a credential.
- Keep one profile per workspace tier (`np`, `prod`). Developer bundle deploys to the `dev` target use the nonprod profile; auth resolves bundle settings, then environment variables, then profiles.
- Human auth is for development in the nonprod workspace. `nonprod` and `prod` targets deploy only from CI as the deploy SP ([Environments](databricks-environments.md)).

## Rules

- Every production workload runs as a service principal. `mode: production` targets set `run_as` to the pipeline or deploy SP; a job owned by a human identity in `nonprod` or `prod` is a defect.
- Pipeline SPs hold no credentials. Data access flows through Unity Catalog storage credentials backed by the access connector's managed identity. A pipeline SP holding its own Azure credential for storage access is a defect.
- One pipeline SP per domain per environment, granted per the [access matrix](../governance/access-model.md).
- Service principals and federation policies are created by Terraform through the account SCIM API. Bundles reference SPs in `run_as` and `permissions`; they never create identities.
- Auth methods, ranked. Use the highest that works:
  1. Workload identity federation (Databricks federation policy or Entra federated credential). No stored credential.
  2. OAuth M2M client secret, stored in a Key Vault-backed secret scope with a rotation owner ([Secrets and credentials](secrets-and-credentials.md)).
  3. Personal access tokens: banned. [Databricks documents PATs as legacy](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/manage-service-principals), for use "only when OAuth is not supported"; no such case exists on this platform.
- Databricks federation policies attach only to Databricks-managed SPs. Policy scoping rules: [CI/CD](github-cicd-and-deployment.md).
- Entra auth types (`azure-cli`, managed identity) are used only by the Terraform identity. Workload SPs authenticate with Databricks OAuth.
- Deactivate before delete. Deactivation blocks authentication and is reversible; deletion fails jobs, stops compute owned by the SP, and breaks Run-as-Owner dashboards until ownership is reassigned.
- On-behalf-of and Databricks Apps user-authorization patterns are not yet written; add them here when the first app ships.

## Sharp edges

- Deactivating an SP blocks authentication but does not revoke tokens: per the [Databricks docs](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/manage-service-principals), "the tokens remain but cannot be used to authenticate while a service principal is deactivated," and they work again on reactivation. Revoke on offboarding; the PAT ban is what makes deactivation airtight here.
- With the `RestrictWorkspaceAdmins` setting at `ALLOW ALL`, workspace admins can mint tokens on behalf of any SP in their workspace. Restrict the setting in every workspace.
- A Databricks federation policy on an Entra-managed SP is undocumented behavior. Do not design around it; deploy SPs stay Databricks-managed.
- Entra SPs synced by automatic identity management provision on first use. A synced SP is invisible to grants and APIs until it first authenticates; grant against it only after provisioning.
- The Entra SCIM provisioning connector cannot sync service principals; only automatic identity management can. A hand-linked application ID is a one-time copy, not a sync.
- Deleting an account-level SP removes it from every workspace at once. There is no workspace-scoped delete for federated workspaces; use workspace-level deactivation instead.

## Checklist

- [ ] Every `nonprod`/`prod` job and pipeline runs as a service principal via `run_as`
- [ ] Deploy and pipeline SPs are Databricks-managed; the Terraform identity is the only Entra-federated identity
- [ ] Zero PATs; every stored credential is an OAuth M2M secret in a Key Vault-backed scope with a rotation owner
- [ ] Every SP and federation policy exists in Terraform state
- [ ] No pipeline SP holds an Azure credential; storage access is via UC storage credentials only
- [ ] `RestrictWorkspaceAdmins` is restricted in every workspace
- [ ] Offboarding an SP: deactivate, reassign ownership, then delete

## Sources

- Azure Databricks: [Databricks CLI authentication](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/cli/authentication)
- Azure Databricks: [Service principals](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals)
- Azure Databricks: [Manage service principals](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/manage-service-principals)
- Azure Databricks: [Enable workload identity federation for GitHub Actions](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/auth/provider-github)
- Azure Databricks: [Automatic identity management](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/automatic-identity-management/)
- Microsoft Entra: [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
