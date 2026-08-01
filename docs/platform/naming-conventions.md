# Naming Conventions

Defines naming standards for every named resource on the platform. Nothing is named ad hoc; examples and templates in this repo must follow these conventions. Names are contracts: a rename after consumers depend on it is a breaking change.

## What this covers

- Semantic naming: names as the machine-readable map of the platform
- Which case style applies in which context
- Name patterns for Unity Catalog objects, Databricks workspace objects, Azure resources, identities, secrets, tags, and repos
- The platform constraints that force these choices

## Names carry meaning

Unity Catalog metadata is the primary map humans and agents have of the data landscape; a name that does not inform misleads both.

- Full words: `customer_acquisition_cost`, not `cust_acq_cst`. Brief and semantic, never cryptic.
- Abbreviate only where a platform limit forces it (the Azure 24-character types: `dbx`, `np`, `[region]`). UC names have a 255-character limit; spell them out.
- Industry-standard abbreviations only: `id`, `qty`, `pct`.
- A name states what a thing is; the COMMENT states what it means. See [Metadata and comments](metadata-and-comments.md).

## Case styles and where they apply

Each style has exactly one home. "Not used" is a rule, not an observation.

| Style | Example | Where it applies here |
| --- | --- | --- |
| `snake_case` | `order_line_id` | All Unity Catalog objects. Python modules, packages, and variables. SQL identifiers. YAML and Terraform keys. |
| `kebab-case` | `data-platform-practices` | Everything Databricks that is not a UC object or Python module: notebooks, folders, jobs, pipelines, bundles and resource keys, warehouses. Azure resource names (where allowed), repos, secret scopes and keys, CLI flags. |
| `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` | Environment variables and code constants only. |
| `camelCase` | `costCenter` | Tag keys and JSON payload fields only. |
| `PascalCase` | `OrderIngestionService` | Class names in application code only. |

The split is not stylistic. It falls out of three hard constraints:

1. Unity Catalog lowercases object names, and hyphens force backtick quoting in every query. UC objects: `snake_case`.
2. Python cannot `import my-module`; `-` parses as minus. Importable modules: `snake_case`. Everything referenced by path: `kebab-case`.
3. Key Vault secret names forbid underscores; storage account names forbid hyphens. Azure: `kebab-case`, bare concatenation where hyphens are illegal.

## Unity Catalog objects

All UC names: `snake_case`, ASCII `[a-z0-9_]`, starting with a letter. UC permits more behind backticks; this platform does not use them.

| Object | Pattern | Example |
| --- | --- | --- |
| Catalog | `<env>_catalog` | `dev_catalog`, `prod_catalog` |
| Schema | medallion layer | `bronze`, `silver`, `gold` |
| Bronze table | `<entity>` | `customer`, `order_line` |
| Silver table | `<domain>` fact or entity | `sales`, `customer` |
| Gold object | `<domain>_<grain>` | `sales_daily`, `sales_product_monthly` |
| Column | `<variable>` | `order_date`, `unit_price` |
| Volume | `<purpose>` | `landing`, `checkpoints` |

Rules:

- No layer prefixes in table or column names (see [Medallion data practices](../practices/medallion-data-practices.md)). The schema tells you the layer; the COMMENT tells you what it is.
- Bronze system columns keep their leading underscore: `_ingest_timestamp`, `_source_file`, `_rescued_data`.
- No abbreviations in column names except industry-standard (`id`, `qty`, `pct`).
- Singular entity names (`customer`, not `customers`); the grain statement says what a row is.
- Every UC object also requires a COMMENT: [Metadata and comments](metadata-and-comments.md).

## Databricks workspace objects

Non-UC assets are referenced by path or ID, never as identifiers: `kebab-case`. Verified: bundle interpolation accepts hyphens in resource keys (`${resources.jobs.my-job.id}` is valid per the [CLI reference grammar](https://github.com/databricks/cli/blob/main/libs/dyn/dynvar/ref.go)); job, pipeline, notebook, and warehouse names have no restriction excluding hyphens.

| Object | Pattern | Example |
| --- | --- | --- |
| Pipeline (Lakeflow SDP) | `<domain>-medallion` | `sales-medallion` |
| Conformed dimension pipeline | `conformed-dimensions` | `conformed-dimensions` |
| Cross-domain Gold job | `<subject>-gold` | `cac-gold` |
| Job | `<domain>-<purpose>` | `sales-maintenance` |
| Bundle (databricks.yml `name`) | matches repo | `sales-pipelines` |
| Bundle resource key | matches the resource name | `sales-maintenance` |
| Notebook (entry point, orchestration, exploration) | `<purpose>` | `daily-load-orchestrator` |
| Notebook or `.py` file imported as a module | `snake_case` (forced) | `date_utils.py` |
| SQL warehouse | `<team>-<size>` | `analytics-small` |
| Cluster policy | `<workload-class>` | `jobs-standard` |

Rules:

- A file consumed via `import` must be a valid Python identifier. If a notebook may ever be imported, name it `snake_case` at creation; renames break job paths.
- A file defining one UC table is named after the table: `sales_daily.sql` defines `gold.sales_daily`.
- Python wheels: distribution name may be kebab (`sales-pipelines`); the import package must be snake (`sales_pipelines`).
- Environment never appears in job, pipeline, or warehouse names; it comes from the bundle target and the catalog it deploys into.
- Job and pipeline names live in bundles, in source control. Names outside bundles drift.

## Azure resources

Pattern: `[subject]-[scope]-<type>[-<purpose>]-[region]-<env>`. Suffix-based: the environment is always the last token, and subject and scope lead, so related resources sort together in any listing and the same resource differs across environments only at the tail.

Standard tokens, defined once:

- `[subject]`: the business subject or product line that owns the resource. Set the real token in Terraform and register it here before first use; docs use the placeholder.
- `[scope]`: the workload within the subject. This platform's core infrastructure: `dbx`. Domain-scoped resources (domain secret vaults) use the domain token as scope.
- `<type>`: readable resource-type token from the table below. Full words where limits allow (`workspace`, `vault`, `log`); abbreviate only where a length limit forces it (`st`).
- Environments (catalogs, bundle targets): `dev`, `nonprod`, `prod`. Workspace tiers: `nonprod`, `prod`, spelled out; abbreviate to `np` only where a length limit forces it.
- Region: examples in this repo use the `[region]` placeholder. Set the real CAF region token in Terraform and register it here before first use; do not hardcode a region in docs.
- No arbitrary instance counters. `001` on a singleton is noise. Append a two-digit series suffix only when an intentional series of the same entity exists, and register the series here.

Azure resources scope to tier, not environment: one nonprod-tier workspace hosts the `dev` and `nonprod` environments (see [Environments](databricks-environments.md)).

| Resource | Type token | Constraint that matters | Example |
| --- | --- | --- | --- |
| Resource group | `rg` | 1-90 chars | `[subject]-dbx-rg-[region]-nonprod` |
| Databricks workspace | `workspace` | 3-64, alphanumerics, underscores, hyphens | `[subject]-dbx-workspace-[region]-nonprod` |
| Databricks access connector | `connector` | | `[subject]-dbx-connector-[region]-nonprod` |
| Storage account | `st` | 3-24, lowercase alphanumeric only, globally unique | `[subject]dbxst[region]np` |
| ADLS container | none | 3-63, lowercase, numbers, hyphens | `bronze-landing`, `uc-managed` |
| Key vault | `vault` | 3-24, alphanumerics and hyphens, globally unique | `[subject]-dbx-vault-[region]-np` |
| Virtual network | `vnet` | 2-64 | `[subject]-dbx-vnet-[region]-nonprod` |
| Subnet | `snet` | | `[subject]-dbx-snet-host-[region]-nonprod` |
| Private endpoint | `endpoint` | | `[subject]-dbx-endpoint-storage-blob-[region]-nonprod` |
| Log Analytics workspace | `log` | | `[subject]-dbx-log-[region]-nonprod` |

Rules:

- Storage accounts and key vaults have 24-character global limits, which is why their examples abbreviate type and environment (`st`, `np`). Verify the budget for the chosen subject and region tokens before first use; storage accounts also forbid hyphens, so their tokens concatenate bare.
- Names are assigned in infrastructure code (see [Azure infrastructure](azure-infrastructure.md)). A portal-created resource with an ad hoc name is two defects, not one.
- The network patterns (`vnet`, `snet`) apply only to a VNet-injected fallback workspace; standard workspaces are serverless and have no customer VNet. `endpoint` also names NCC private endpoints to protected resources.
- Identity objects (service principals, groups, managed identities) are the exception to suffix style: they keep kind prefixes (next section), because identities are found by kind first.

## Identity

Identity keeps kind prefixes (`sp-`, `grp-`, `id-`), unlike Azure resources: identities are found by kind first, so sorting by prefix is the useful order.

| Object | Pattern | Example |
| --- | --- | --- |
| Pipeline service principal | `sp-<domain>-pipeline-<env>` | `sp-sales-pipeline-prod` |
| Cross-domain Gold job service principal | `sp-<subject>-gold-<env>` | `sp-cac-gold-prod` |
| Deployment service principal | `sp-dbx-deploy-<tier>` | `sp-dbx-deploy-np`, `sp-dbx-deploy-prod` |
| App service principal | `sp-<app>-<env>` | `sp-salesapi-prod` |
| Entra / UC group | `grp-<role>-<scope>-<env>` | `grp-analysts-gold-prod`, `grp-dataeng-silver-dev` |
| Managed identity | `id-<scope>-<purpose>-<tier>` | `id-dbx-deploy-np` |

Grants attach to groups, never users (see [Access model](../governance/access-model.md)). Group names encode role and scope so a grant audits from the name alone.

## Secrets

| Object | Pattern | Example |
| --- | --- | --- |
| Secret scope | `<domain>-<env>` | `sales-prod` |
| Backing key vault | `[subject]-<domain>-vault-[region]-<env>` | `[subject]-sales-vault-[region]-prod` |
| Secret key | `<system>-<credential>` | `salesforce-client-secret` |

Key Vault secret names permit only alphanumerics and hyphens; `kebab-case` is forced. No environment or hostname in key names; the scope carries the environment.

Every secret scope has its own backing vault, 1:1 (see [Secrets and credentials](secrets-and-credentials.md)). The 24-character vault budget breaks on long domain or subject tokens: abbreviate `nonprod` to `np` only where forced, and register any shortened token here before first use. The platform vault `[subject]-dbx-vault-[region]-<tier>` holds platform secrets only, never domain scope secrets.

## Tags

Tag keys: `camelCase`. Tag values: freeform but stable; use standard tokens where one exists.

Baseline keys on every Azure and Databricks compute resource: `env`, `domain`, `owner`, `costCenter`, `managedBy` (`terraform` or `bundle`). Enforcement and additional required tags: [Compute policies](databricks-compute-policies.md).

Azure stores tag keys as first written; two spellings of one key (`CostCenter`, `costcenter`) fragment cost reports.

## Repos and files

- Repos and bundles: `kebab-case` (`sales-pipelines`, `data-platform-practices`).
- Python modules and files: `snake_case`.
- A file defining one table is named after the table: `sales_daily.sql` defines `gold.sales_daily`.
- Commit messages, code comments, docstrings: [Metadata and comments](metadata-and-comments.md).

## Sharp edges

- Unity Catalog lowercases object names: `SalesDaily` silently becomes `salesdaily`. Any casing scheme but `snake_case` is destroyed on write.
- A hyphenated UC name works behind backticks, then requires them in every query and tool forever. Treat it as a defect even though the platform accepts it.
- Column names with special characters require Delta column mapping and break downstream tools. Stay in `[a-z0-9_]`.
- A kebab notebook fails at `import` with a syntax error; the fix is a rename that breaks every referencing job path. Decide import-vs-entry-point at creation.
- Storage account and key vault names are globally unique; a reviewed name can still fail at deploy. Check availability in the infrastructure pipeline, not manually.
- The 24-character storage budget is consumed by tokens; lengthening the subject or scope token, or adding one, breaks the one resource type that cannot take hyphens.
- Environment in a job or pipeline name means promotion renames it, orphaning run history and breaking references.
- Renaming a UC table does not update downstream views, dashboards, or retrieval configs. Fix bad names immediately or live with them.

## Checklist

- [ ] Every UC object name matches `[a-z][a-z0-9_]*` and its pattern above
- [ ] No abbreviations outside the Azure-forced tokens and the industry-standard set
- [ ] No layer prefix in any table or column name
- [ ] No environment token in any job, pipeline, or warehouse name
- [ ] Importable Python files and notebooks are `snake_case`
- [ ] Every Azure resource name is generated in infrastructure code from the standard tokens
- [ ] Environment is the final token of every Azure resource name; no instance counters without a registered series
- [ ] Storage account and key vault names fit the 24-character budget
- [ ] Every grant-bearing group name encodes role, scope, and environment
- [ ] Tag keys are `camelCase` and include the baseline set
- [ ] New name patterns are added to this doc before first use

## Sources

- Databricks: [Names and identifiers](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-names)
- Databricks CLI source, bundle reference grammar: [libs/dyn/dynvar/ref.go](https://github.com/databricks/cli/blob/main/libs/dyn/dynvar/ref.go)
- Python Language Reference: [Identifiers and keywords](https://docs.python.org/3/reference/lexical_analysis.html#identifiers)
- Microsoft Learn: [Naming rules and restrictions for Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules)
- Microsoft Learn (Cloud Adoption Framework): [Abbreviation recommendations for Azure resources](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
