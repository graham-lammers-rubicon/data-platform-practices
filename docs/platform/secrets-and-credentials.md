# Secrets and Credentials

Defines how secrets are stored, accessed, rotated, and kept out of repos. The standing rule: no secrets in code, notebooks, or docs; docs never contain real hostnames, tokens, or workspace IDs.

## What this covers

- The storage hierarchy: when no secret is needed, and where required secrets live
- Key Vault-backed secret scope rules
- Repo hygiene: keeping secrets out of GitHub and Terraform state
- Rotation and expiration

## Storage hierarchy

Prefer the highest option that works:

1. **No secret.** Workload identity federation for CI ([CI/CD](cicd-and-deployment.md)), Unity Catalog storage and service credentials backed by access connector managed identities for Azure services. Databricks documents managed identities as ["strongly recommended"](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-services/service-credentials) over service principals: they work behind network rules and "remove the need to manage and rotate secrets."
2. **Azure Key Vault-backed secret scope.** The only storage for secrets that must exist: third-party system credentials (`salesforce-client-secret`), OAuth M2M client secrets ([Service principal authentication](service-principal-auth.md)). The scope is a read-only interface; the secret lives in the vault.
3. **Databricks-backed scopes: not used.** They store secrets in a Databricks-managed database outside the tenant's vault governance.

Watch item, not normative: Unity Catalog secrets (`catalog.schema.secret`, governed by UC privileges, audited in `system.access.audit`) are Public Preview with hard limits: no SQL warehouses, DBR 17.3+ or serverless environment 4+ only, no information schema, 100 secrets per schema. Revisit at GA; the UC governance model would supersede scope ACLs.

## Rules

- Every stored secret lives in Azure Key Vault and is surfaced to Databricks through a Key Vault-backed secret scope. `databricks secrets list-scopes` showing a scope without a Key Vault DNS name is a defect.
- One vault per scope, 1:1. Scope ACLs are scope-level, and a scope grants access to every secret in its backing vault; the vault is therefore the access boundary. Scope `<domain>-<env>` is backed by vault `kv-<domain>-<env>-[region]-001` ([Naming conventions](naming-conventions.md)).
- Vaults backing scopes use the **Vault access policy** permission model. Per the [Databricks secrets doc](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/), "Azure role-based access control (RBAC) is not supported" for Key Vault-backed scopes.
- Vaults, scopes, and scope ACLs are Terraform-created, per the everything-as-code rule ([Environments](environments.md)). Scope ACLs grant READ to the consuming pipeline SP or group only; MANAGE stays with the platform team.
- Secret writes go through Azure (portal, API, or Terraform), never through Databricks; the scope is read-only. Rotation happens in Key Vault and is invisible to consuming code.
- Every secret has an expiration date and a named rotation owner.
- Code reads secrets with `dbutils.secrets.get()` or the `{{secrets/<scope>/<key>}}` Spark configuration reference. A secret value assigned, transformed, logged, or written to a table is a defect regardless of redaction.
- CI holds no Databricks credentials: deploy auth is OIDC federation, which Databricks documents as ["the most secure way to authenticate"](https://docs.databricks.com/aws/en/dev-tools/bundles/ci-cd-bundles) for CI/CD. GitHub Actions secrets store nothing that Key Vault or federation can carry; they are CI plumbing, not a secret store, invisible to Databricks workloads and without expiration or rotation.
- A workflow that genuinely needs a stored secret fetches it from Key Vault at runtime: `azure/login` with OIDC, `az keyvault secret show`, then `::add-mask::` before use. Key Vault stays the single source of truth for both Databricks and CI. The values GitHub stores for this (client, tenant, subscription IDs, vault name) are identifiers, not credentials.
- A vault that backs a Databricks scope is on the access-policy model; grant the CI identity a Get access policy on that vault, not an RBAC role. Grants are per vault, least privilege, same as scope ACLs.
- Repos never contain secrets. GitHub Secret Protection with push protection is enabled on every repo; a bypass requires a recorded reason and raises an audited alert.
- A secret that reaches a commit is compromised the moment it is pushed. Rotate it first; scrubbing history is cleanup, not remediation.
- Terraform never places secret values in configuration; state stores them in plain text. Terraform creates the vault, access policies, and scope; values are set out-of-band. State is remote, encrypted, and access-controlled per [HashiCorp guidance](https://developer.hashicorp.com/terraform/language/state/sensitive-data): "treat your state file as sensitive data."

## Sharp edges

- A vault on the Azure RBAC permission model cannot back a secret scope; the scope creation fails. Switching the model changes who can access the vault; plan it, don't toggle it.
- Scope access is vault access. Granting READ on a scope exposes every secret in the backing vault, which is why the 1:1 vault-per-scope rule exists. A shared "common" vault is a cross-domain leak.
- Redaction replaces only literal values with `[REDACTED]`. Any transformation of the secret prints plainly; the [Databricks secrets doc](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/) is explicit that redaction "does not prevent deliberate and arbitrary transformations." ACLs are the control, not redaction.
- Workspace admins, scope creators, and granted users can read secret contents; [Databricks states](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/) it "is not possible to fully prevent these users from viewing secret contents." Grant accordingly.
- Creating a Key Vault-backed scope requires Key Vault Contributor, Contributor, or Owner on the vault, even when Databricks already has data-plane access.
- Scope creation grants Get/List on the vault to the Azure Databricks first-party application via access policy. The vault firewall needs the trusted-Microsoft-services bypass when locked down.
- The 24-character Key Vault name budget breaks for long domain tokens (with a three-character region token, `kv-marketing-nonprod-[region]-001` resolves to 28). Check the budget per name; abbreviate the environment to `np` only where forced, and register any shortened domain token in the naming conventions before first use.
- `terraform output -json` and `-raw` print values marked `sensitive` in plain text.
- Push protection is off by default and, for private repos, requires the GitHub Secret Protection license on Team or Enterprise. An unlicensed private repo has no push-time guard; the rule still holds, enforcement is just weaker.
- GitHub-hosted runners run on GitHub's infrastructure, outside the platform VNet. A vault locked to private endpoints is unreachable from them; the runtime-fetch pattern needs the vault firewall to admit the runner, or self-hosted runners in the VNet. Verify reachability before making a workflow depend on it.
- The runtime-fetch pattern masks manually: a fetched value used before `::add-mask::` prints in the log. Mask immediately after retrieval, always.

## Checklist

- [ ] Every secret scope is Key Vault-backed; zero Databricks-backed scopes exist
- [ ] Every backing vault: access-policy model, 1:1 with its scope, defined in Terraform
- [ ] Scope ACLs grant READ to consuming SPs or groups only; MANAGE held by the platform team
- [ ] Every secret has an expiration date and rotation owner
- [ ] No secret value appears in Terraform configuration; state is remote, encrypted, access-controlled
- [ ] Push protection enabled on every repo; secret scanning alerts triaged, not ignored
- [ ] Any pushed secret was rotated before history cleanup
- [ ] Docs and examples use placeholder values only

## Sources

- Azure Databricks: [Secret management](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/)
- Azure Databricks: [Secrets in Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/unity-catalog-secrets)
- Azure Databricks: [Create service credentials](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-services/service-credentials)
- GitHub: [About secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning)
- GitHub: [About push protection](https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection)
- GitHub: [Secrets in GitHub Actions](https://docs.github.com/en/actions/concepts/security/secrets) and [secrets reference](https://docs.github.com/en/actions/reference/security/secrets)
- Microsoft Learn: [Use Azure Key Vault secrets in a GitHub Actions workflow](https://learn.microsoft.com/en-us/azure/developer/github/github-actions-key-vault)
- Databricks: [CI/CD on Databricks](https://docs.databricks.com/aws/en/dev-tools/bundles/ci-cd-bundles) (federation "eliminates the need for Databricks secrets")
- HashiCorp: [Sensitive data in Terraform state](https://developer.hashicorp.com/terraform/language/state/sensitive-data)
