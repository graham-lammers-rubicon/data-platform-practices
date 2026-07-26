# Naming Conventions

Defines naming standards for every named resource on the platform. Nothing is named ad hoc; examples and templates in this repo must follow these conventions. Names are contracts: a rename after consumers depend on it is a breaking change.

## What this covers

- Semantic naming: names and comments as the machine-readable map of the platform
- Which case style applies in which context
- Name patterns for Unity Catalog objects, Databricks workspace objects, Azure resources, identities, secrets, tags, and repos
- The platform constraints that force these choices

## Names carry meaning

Names are read far more often than written, by humans and by agents. Unity Catalog metadata is the primary map an agent has of the data landscape; a name that does not inform misleads both.

- Full words: `customer_acquisition_cost`, not `cust_acq_cst`. Brief and semantic, never cryptic. Casual abbreviations do not inform.
- Compressed tokens exist only where a platform limit forces them (the Azure 24-character types: `dplat`, `np`, `wus`). UC names have a 255-character limit and no budget pressure; never compress them.
- Industry-standard abbreviations only: `id`, `qty`, `pct`.
- A name states what a thing is; the COMMENT states what it means (grain, unit, scope). Both are required; neither substitutes for the other.

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
| Gold object | `<domain>_<grain>` | `sales_daily`, `customer_monthly` |
| Column | `<variable>` | `order_date`, `unit_price` |
| Volume | `<purpose>` | `landing`, `checkpoints` |

Rules:

- No layer prefixes in table or column names (see [Medallion data practices](../practices/medallion-data-practices.md)). The schema tells you the layer; the COMMENT tells you what it is.
- Bronze system columns keep their leading underscore: `_ingest_timestamp`, `_source_file`, `_pipeline_id`, `_is_quarantined`, `_raw_payload`.
- No abbreviations in column names except industry-standard (`id`, `qty`, `pct`).
- Singular entity names (`customer`, not `customers`); the grain statement says what a row is.
- Every UC object carries a COMMENT: catalogs and schemas state scope and owner; tables state grain and content (see [Medallion data practices](../practices/medallion-data-practices.md)); columns state meaning, unit, and semi-additive labeling where it applies. An object without a COMMENT is incomplete.
- COMMENTs are queryable metadata (`INFORMATION_SCHEMA`, system tables). Humans and agents discover the landscape through them, not tribal knowledge; an uncommented object is invisible to both.

## Databricks workspace objects

Non-UC assets are referenced by path or ID, never as identifiers: `kebab-case`. Verified: bundle interpolation accepts hyphens in resource keys (`${resources.jobs.my-job.id}` is valid per the CLI reference grammar); job, pipeline, notebook, and warehouse names have no restriction excluding hyphens.

| Object | Pattern | Example |
| --- | --- | --- |
| Pipeline (Lakeflow SDP) | `<domain>-medallion` | `sales-medallion` |
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

Pattern: `<abbrev>-<workload>-<env>-<region>-<instance>`, using Cloud Adoption Framework abbreviations. Workload for this platform: `dplat`.

Standard tokens, defined once:

- Environments (catalogs, bundle targets): `dev`, `qa`, `test`, `prod`
- Workspace tiers (Azure resources, workspaces): `np` (nonprod), `prod`
- Regions: `wus` (West US); add tokens as regions are added
- Instance: `001`, incremented only when a second instance exists

Azure resources scope to tier, not environment: one nonprod workspace hosts `dev`, `qa`, and `test` (see [Environments](environments.md)).

| Resource | Abbrev | Constraint that matters | Example |
| --- | --- | --- | --- |
| Resource group | `rg` | 1-90 chars | `rg-dplat-np-wus-001` |
| Databricks workspace | `dbw` | 3-64, alphanumerics, underscores, hyphens | `dbw-dplat-np-wus-001` |
| Databricks access connector | `dbac` | | `dbac-dplat-np-wus-001` |
| Storage account | `st` | 3-24, lowercase alphanumeric only, globally unique | `stdplatnpwus001` |
| ADLS container | none | 3-63, lowercase, numbers, hyphens | `bronze-landing`, `uc-managed` |
| Key vault | `kv` | 3-24, alphanumerics and hyphens, globally unique | `kv-dplat-np-wus-001` |
| Virtual network | `vnet` | 2-64 | `vnet-dplat-np-wus-001` |
| Subnet | `snet` | | `snet-dbw-private-001` |
| Private endpoint | `pep` | | `pep-stdplatnpwus001-blob` |
| Log Analytics workspace | `log` | | `log-dplat-np-wus-001` |
| Managed identity | `id` | | `id-dplat-deploy-np-001` |

Rules:

- Storage accounts and key vaults have 24-character global limits. The token set fits; re-check the budget before lengthening any token.
- Names are assigned in infrastructure code (see [Azure infrastructure](azure-infrastructure.md)). A portal-created resource with an ad hoc name is two defects, not one.

## Identity

| Object | Pattern | Example |
| --- | --- | --- |
| Pipeline service principal | `sp-<domain>-pipeline-<env>` | `sp-sales-pipeline-prod` |
| Deployment service principal | `sp-dplat-deploy-<env>` | `sp-dplat-deploy-prod` |
| App service principal | `sp-<app>-<env>` | `sp-salesapi-prod` |
| Entra / UC group | `grp-<role>-<scope>-<env>` | `grp-analysts-gold-prod`, `grp-dataeng-dev` |

Grants attach to groups, never users (see [Access model](../governance/access-model.md)). Group names encode role and scope so a grant audits from the name alone.

## Secrets

| Object | Pattern | Example |
| --- | --- | --- |
| Secret scope | `<domain>-<env>` | `sales-prod` |
| Secret key | `<system>-<credential>` | `salesforce-client-secret` |

Key Vault secret names permit only alphanumerics and hyphens; `kebab-case` is forced. No environment or hostname in key names; the scope carries the environment.

## Tags

Tag keys: `camelCase`. Tag values: freeform but stable; use standard tokens where one exists.

Baseline keys on every Azure and Databricks compute resource: `env`, `domain`, `owner`, `costCenter`, `managedBy` (`terraform` or `bundle`). Enforcement and additional required tags: [Compute policies](compute-policies.md).

Azure stores tag keys as first written; two spellings of one key (`CostCenter`, `costcenter`) fragment cost reports.

## Repos, files, and commits

- Repos and bundles: `kebab-case` (`sales-pipelines`, `data-platform-practices`).
- Python modules and files: `snake_case`.
- A file defining one table is named after the table: `sales_daily.sql` defines `gold.sales_daily`.
- Commit messages are part of the semantic record: imperative subject stating what changed, body stating why. History is how humans and agents reconstruct intent; `fix`, `wip`, and `updates` destroy it.

## Sharp edges

- Unity Catalog lowercases object names: `SalesDaily` silently becomes `salesdaily`. Any casing scheme but `snake_case` is destroyed on write.
- A hyphenated UC name works behind backticks, then requires them in every query and tool forever. Treat it as a defect even though the platform accepts it.
- Column names with special characters require Delta column mapping and break downstream tools. Stay in `[a-z0-9_]`.
- A kebab notebook fails at `import` with a syntax error; the fix is a rename that breaks every referencing job path. Decide import-vs-entry-point at creation.
- Storage account and key vault names are globally unique; a reviewed name can still fail at deploy. Check availability in the infrastructure pipeline, not manually.
- The 24-character storage budget is consumed by tokens; lengthening `dplat` or adding a token breaks the one resource type that cannot take hyphens.
- Environment in a job or pipeline name means promotion renames it, orphaning run history and breaking references.
- Renaming a UC table does not update downstream views, dashboards, or retrieval configs. Fix bad names immediately or live with them.

## Checklist

- [ ] Every UC object name matches `[a-z][a-z0-9_]*` and its pattern above
- [ ] Every UC object has a COMMENT; tables declare grain, columns declare meaning and unit
- [ ] No compressed names outside the Azure-forced tokens
- [ ] No layer prefix in any table or column name
- [ ] No environment token in any job, pipeline, or warehouse name
- [ ] Importable Python files and notebooks are `snake_case`
- [ ] Every Azure resource name is generated in infrastructure code from the standard tokens
- [ ] Storage account and key vault names fit the 24-character budget
- [ ] Every grant-bearing group name encodes role, scope, and environment
- [ ] Tag keys are `camelCase` and include the baseline set
- [ ] New name patterns are added to this doc before first use

## Sources

- Databricks: [Names and identifiers](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-names)
- Databricks CLI source, bundle reference grammar: [libs/dyn/dynvar/ref.go](https://github.com/databricks/cli/blob/main/libs/dyn/dynvar/ref.go)
- Python Language Reference: [Identifiers and keywords](https://docs.python.org/3/reference/lexical_analysis.html#identifiers)
- Microsoft Learn: [Naming rules and restrictions for Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules)
- Microsoft Learn (Cloud Adoption Framework): [Abbreviation recommendations for Azure resources](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
