# Platform

Infrastructure reference for the Azure-hosted Databricks platform. Owned by platform engineering; consulted by everyone. Files are prefixed by system (`azure-`, `databricks-`, `github-`) where one system owns the doc; unprefixed docs are cross-system standards.

Standing the platform up from zero? Read in this order: Azure infrastructure → Databricks environments → GitHub CI/CD → Databricks compute policies → secrets and service principal auth. The rest are consulted as needed.

## Pages

- [Azure infrastructure](azure-infrastructure.md) - serverless workspaces, NCC connectivity, Unity Catalog wiring, identity, Terraform standard
- [Databricks compute policies](databricks-compute-policies.md) - workload classes, sizing, termination, cost attribution
- [Databricks environments](databricks-environments.md) - two workspace tiers, environment-per-catalog, one-way cross-tier access, bundle-only promotion
- [Databricks Genie spaces](databricks-genie-spaces.md) - spaces as code: scope, trusted assets, benchmarks, certification wiring
- [Databricks metric views](databricks-metric-views.md) - metric views as the governed metric implementation
- [Databricks service principal authentication](databricks-service-principal-auth.md) - SP types, standard identities, auth ranking, lifecycle
- [GitHub CI/CD and deployment](github-cicd-and-deployment.md) - GitHub Actions pipeline, promotion gates, OIDC deployment identity, rollback
- [Naming conventions](naming-conventions.md) - case styles, name patterns, and tokens for UC objects, Databricks assets, Azure resources, identities, secrets, and tags
- [Metadata and comments](metadata-and-comments.md) - UC COMMENT requirements, commit message standard, Python docstrings
- [Secrets and credentials](secrets-and-credentials.md) - storage hierarchy, Key Vault-backed scopes, repo hygiene
- [Resilience and disaster recovery](resilience.md) - stub: DR/BCP decision register, owners open
