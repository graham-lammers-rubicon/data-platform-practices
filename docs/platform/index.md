# Platform

Infrastructure reference for the Azure-hosted Databricks platform. Owned by platform engineering; consulted by everyone.

## Pages

- [Azure infrastructure](azure-infrastructure.md) - resource layout, VNet injection, Unity Catalog wiring, Terraform standard
- [Environments](environments.md) - two workspace tiers, environment-per-catalog, one-way cross-tier access, bundle-only promotion
- [CI/CD and deployment](cicd-and-deployment.md) - GitHub Actions pipeline, promotion gates, OIDC deployment identity, rollback
- [Compute policies](compute-policies.md) - workload classes, sizing, termination, cost attribution
- [Naming conventions](naming-conventions.md) - case styles, name patterns, and tokens for UC objects, Databricks assets, Azure resources, identities, secrets, and tags
- [Metadata and comments](metadata-and-comments.md) - UC COMMENT requirements, commit message standard, Python docstrings
- [Secrets and credentials](secrets-and-credentials.md) - storage hierarchy, Key Vault-backed scopes, repo hygiene
- [Service principal authentication](service-principal-auth.md) - SP types, standard identities, auth ranking, lifecycle
