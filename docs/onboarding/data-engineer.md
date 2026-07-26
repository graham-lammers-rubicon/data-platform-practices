# Onboarding: Data Engineer

You build and operate the pipelines that move data from source through Bronze, Silver, and Gold. This page is your reading order and first-week checklist.

## Read in this order

1. [Access model](../governance/access-model.md) - what you can touch and why the layer boundaries are hard rules
2. [Medallion data practices](../practices/medallion-data-practices.md) - the core reference: layer contracts, SCD2 in Bronze, pipeline architecture
3. [Tidy data](../practices/tidy-data.md) - the shape rules for Silver and why Gold is wide
4. [Analytical dataset language](../practices/analytical-dataset-language.md) - grain, measures vs. metrics; the contract you publish against
5. [Spec-driven development](../practices/spec-driven-development.md) - no pipeline work starts without a spec
6. Platform reference as needed: [Environments](../platform/environments.md), [Compute policies](../platform/compute-policies.md), [Naming conventions](../platform/naming-conventions.md), [CI/CD and deployment](../platform/cicd-and-deployment.md), [Secrets and credentials](../platform/secrets-and-credentials.md)

## First-week checklist

- [ ] Workspace access granted per the [access model](../governance/access-model.md)
- [ ] Repo cloned; dev environment reachable per the [environment guide](../platform/environments.md)
- [ ] Read the medallion reference end to end, including the sharp edges sections
- [ ] Walk one existing domain pipeline from Bronze table to Gold object and identify the grain declared at each step
- [ ] Ship one small change through the full spec, review, and deployment path

## Rules you will be held to

- Bronze captures everything and transforms nothing; `rescuedDataColumn` is always configured.
- SCD Type 2 lives in Bronze. Silver reads the versioned entity; it never reconstructs history.
- Never write to Silver directly from source. Never let a consumer touch Bronze or Silver.
- Grain is declared in the table COMMENT before schema work starts.
- All writes route through Unity Catalog managed tables via Lakeflow; writes outside it create lineage gaps.
