# Platform

Infrastructure reference for the Azure-hosted Databricks platform. Owned by platform engineering; consulted by everyone.

## Pages

- [Azure infrastructure](azure-infrastructure.md) - subscriptions, resource groups, workspaces, storage, networking, identity *(stub)*
- [Environments](environments.md) - two workspace tiers, environment-per-catalog, one-way cross-tier access, bundle-only promotion
- [CI/CD and deployment](cicd-and-deployment.md) - bundles, pipelines, deployment identity, rollback *(stub)*
- [Compute policies](compute-policies.md) - workload classes, sizing, termination, cost attribution *(stub)*
- [Naming conventions](naming-conventions.md) - case styles, name patterns, and tokens for UC objects, Databricks assets, Azure resources, identities, secrets, and tags
- [Secrets and credentials](secrets-and-credentials.md) - Key Vault-backed scopes, rotation, integration patterns *(stub)*
- [Service principal authentication](service-principal-auth.md) - SP types, least-privilege grants, app auth *(stub)*
