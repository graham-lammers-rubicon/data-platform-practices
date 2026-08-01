# Platform

Infrastructure reference for the Azure-hosted Databricks platform. Owned by platform engineering; consulted by everyone. Files are prefixed by system (`azure-`, `databricks-`, `github-`) where one system owns the doc; unprefixed docs are cross-system standards.

Use the groups, not the file list: standing the platform up, read group 1 top to bottom; securing workloads, group 2; serving BI and agents, group 3; groups 4 and 5 apply to every change.

## 1. Stand up the platform

Read in this order; each builds on the one before.

- [Azure infrastructure](azure-infrastructure.md) - serverless workspaces, NCC connectivity, Unity Catalog wiring, identity, Terraform standard. The footprint everything else deploys into, plus the open network decisions owned with Cloud/DevOps.
- [Databricks environments](databricks-environments.md) - two workspace tiers, environment-per-catalog, one-way cross-tier access, bundle-only promotion. How dev, nonprod, and prod actually differ.
- [GitHub CI/CD and deployment](github-cicd-and-deployment.md) - the pipeline that moves code through those environments: promotion gates, OIDC identity, rollback.
- [Databricks compute policies](databricks-compute-policies.md) - what compute may run, how it terminates, and how every DBU traces to an owner. Includes individual cost discretion.

## 2. Identity and secrets

The security spine; with serverless workspaces there is no network perimeter to lean on.

- [Databricks service principal authentication](databricks-service-principal-auth.md) - machine identities, auth ranking, lifecycle, and human OAuth profiles.
- [Secrets and credentials](secrets-and-credentials.md) - the storage hierarchy: prefer no secret at all; Key Vault-backed scopes for the rest; repo hygiene.

## 3. Serve the semantic layer

The implementation of the [practices arc's](../practices/index.md) serve stage.

- [Databricks metric views](databricks-metric-views.md) - the governed metric implementation: one YAML definition, safe re-aggregation.
- [Databricks Genie spaces](databricks-genie-spaces.md) - natural-language access on top of metric views: spaces as code, trusted assets, benchmarks.

## 4. Standards for every change

- [Naming conventions](naming-conventions.md) - case styles, name patterns, and tokens for everything with a name. Check before creating anything.
- [Metadata and comments](metadata-and-comments.md) - UC COMMENT requirements, commit message standard, docstrings. What makes the platform self-describing.

## 5. Open decision registers

- [Resilience and disaster recovery](resilience.md) - stub: DR/BCP decisions with owners open. Companion register: [data lifecycle](../governance/data-lifecycle.md) in governance.
