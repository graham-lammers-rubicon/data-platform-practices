# Naming Conventions

Defines naming standards for every named resource on the platform. Nothing is named ad hoc; examples and templates elsewhere in this repo must follow these conventions. Names are contracts: renaming a catalog, pipeline, or storage account after consumers depend on it is a breaking change, so names are chosen by rule, once.

## What this covers

- Which case style applies in which context, and why
- Name patterns for Unity Catalog objects, Databricks workspace objects, Azure resources, identities, secrets, tags, and repos
- The platform constraints that force some of these choices

## Case styles and where they apply

Five standard case styles exist in software. On this platform each has exactly one home. Where a style is listed as "not used," that is a rule, not an observation.

| Style | Example | Where it applies here |
| --- | --- | --- |
| `snake_case` | `order_line_id` | All Unity Catalog objects: catalogs, schemas, tables, views, columns. Job and pipeline names. Python modules and variables. SQL identifiers. YAML and Terraform keys. |
| `kebab-case` | `data-platform-practices` | Azure resource names (where allowed), repos, bundle names, secret scopes and secret keys, CLI flags. |
| `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` | Environment variables and code constants only. Never in data object names. |
| `camelCase` | `costCenter` | Tag keys and JSON payload fields only. |
| `PascalCase` | `OrderIngestionService` | Class names in application code only. Never in platform resource names. |

The split is not stylistic. It falls out of two hard constraints:

1. Unity Catalog stores object names as lowercase, and names containing hyphens require backtick quoting in every SQL statement. So UC objects get `snake_case`: casing would be silently destroyed, and hyphens would tax every query forever.
2. Key Vault secret names allow alphanumerics and hyphens only; underscores are invalid. Storage account names allow lowercase letters and numbers only. So the Azure side gets `kebab-case`, degrading to bare concatenation where hyphens are also forbidden.

## Unity Catalog objects

All UC names: `snake_case`, ASCII `[a-z0-9_]` only, starting with a letter. UC permits more characters behind backticks; this platform does not use them. Max length is 255 characters; stay far below it.

| Object | Pattern | Example |
| --- | --- | --- |
| Catalog | `<env>_catalog` | `dev_catalog`, `prd_catalog` |
| Schema | medallion layer | `bronze`, `silver`, `gold` |
| Bronze table | `<entity>` | `customer`, `order_line` |
| Silver table | `<domain>` fact or entity | `sales`, `customer` |
| Gold object | `<domain>_<grain>` | `sales_daily`, `customer_monthly` |
| Column | `<variable>` in snake_case | `order_date`, `unit_price` |
| Volume | `<purpose>` | `landing`, `checkpoints` |

Rules:

- No layer prefixes in table or column names (see [Medallion data practices](../practices/medallion-data-practices.md)). The schema tells you the layer; the COMMENT tells you what it is.
- Bronze system columns keep their leading underscore: `_ingest_timestamp`, `_source_file`, `_pipeline_id`, `_is_quarantined`, `_raw_payload`.
- No abbreviations in column names except industry-standard ones (`id`, `qty`, `pct`). `cust_nm` saves nothing and costs every reader.
- Singular entity names (`customer`, not `customers`). The grain statement, not the plural, says what a row is.

## Databricks workspace objects

| Object | Pattern | Example |
| --- | --- | --- |
| Pipeline (Lakeflow SDP) | `<domain>_medallion` | `sales_medallion` |
| Cross-domain Gold job | `<subject>_gold` | `cac_gold` |
| Job | `<domain>_<purpose>` | `sales_maintenance` |
| Bundle (databricks.yml `name`) | `kebab-case`, matches repo | `sales-pipelines` |
| SQL warehouse | `<team>_<size>` | `analytics_small` |
| Cluster policy | `<workload_class>` | `jobs_standard` |

Rules:

- Environment never appears in job, pipeline, or warehouse names. Environment is expressed by the bundle target and the catalog it deploys into. Embedding `dev` in a name forces a rename at every promotion, which breaks references; deriving it from the target does not.
- Jobs and pipelines are deployed from bundles, so their names live in source control. A name not in a bundle is a name that will drift.

## Azure resources

Pattern: `<abbrev>-<workload>-<env>-<region>-<instance>`, using the Cloud Adoption Framework abbreviations. Workload for this platform is `dataplat`.

Standard tokens, defined once and reused everywhere:

- Environments: `dev`, `stg`, `prd`
- Regions: `eus2` (East US 2); add tokens here as regions are added
- Instance: `001`, incremented only when a second instance genuinely exists

| Resource | Abbrev | Constraint that matters | Example |
| --- | --- | --- | --- |
| Resource group | `rg` | 1-90 chars | `rg-dataplat-dev-eus2-001` |
| Databricks workspace | `dbw` | 3-64, alphanumerics, underscores, hyphens | `dbw-dataplat-dev-eus2-001` |
| Databricks access connector | `dbac` | | `dbac-dataplat-dev-eus2-001` |
| Storage account | `st` | 3-24, lowercase letters and numbers only, globally unique | `stdataplatdeveus2001` |
| ADLS container | none | 3-63, lowercase, numbers, hyphens | `bronze-landing`, `uc-managed` |
| Key vault | `kv` | 3-24, alphanumerics and hyphens, globally unique | `kv-dataplat-dev-eus2-001` |
| Virtual network | `vnet` | 2-64 | `vnet-dataplat-dev-eus2-001` |
| Subnet | `snet` | | `snet-dbw-private-001` |
| Private endpoint | `pep` | | `pep-stdataplatdeveus2001-blob` |
| Log Analytics workspace | `log` | | `log-dataplat-dev-eus2-001` |
| Managed identity | `id` | | `id-dataplat-deploy-dev-001` |

Rules:

- Storage accounts and key vaults have 24-character global limits. The token set above fits; do not invent longer workload or region tokens without re-checking the budget.
- Names are assigned in infrastructure code (see [Azure infrastructure](azure-infrastructure.md)). A portal-created resource with an ad hoc name is two defects, not one.

## Identity

| Object | Pattern | Example |
| --- | --- | --- |
| Pipeline service principal | `sp-<domain>-pipeline-<env>` | `sp-sales-pipeline-prd` |
| Deployment service principal | `sp-dataplat-deploy-<env>` | `sp-dataplat-deploy-prd` |
| App service principal | `sp-<app>-<env>` | `sp-salesapi-prd` |
| Entra / UC group | `grp-<role>-<scope>-<env>` | `grp-analysts-gold-prd`, `grp-dataeng-dev` |

Grants attach to groups, never to individual users (see [Access model](../governance/access-model.md)). Group names therefore encode role and scope, so a grant can be audited from the name alone.

## Secrets

| Object | Pattern | Example |
| --- | --- | --- |
| Secret scope | `<domain>-<env>` | `sales-prd` |
| Secret key | `<system>-<credential>` | `salesforce-client-secret` |

Key Vault secret names permit only alphanumerics and hyphens, so `kebab-case` is mandatory here, not a preference. Never encode the secret's value or environment-specific hostnames in the key name; the scope carries the environment.

## Tags

Tag keys: `camelCase`. Tag values: freeform but stable; use the standard tokens where one exists.

Baseline keys on every Azure resource and Databricks compute resource: `env`, `domain`, `owner`, `costCenter`, `managedBy` (value: `terraform` or `bundle`). The enforcement mechanism and any additional required tags live in [Compute policies](compute-policies.md).

Azure tag names are case-insensitive for operations but stored as first written; two spellings of one key (`CostCenter`, `costcenter`) will fragment cost reports. The `camelCase` rule exists to make that impossible.

## Repos and files

- Repos and bundles: `kebab-case` (`sales-pipelines`, `data-platform-practices`).
- Python modules and files: `snake_case`.
- A SQL or Python file that defines one table is named after the table: `sales_daily.sql` defines `gold.sales_daily`.

## Sharp edges

- Unity Catalog lowercases object names. `SalesDaily` and `sales_daily` are not two naming options; the first silently becomes `salesdaily`. Any casing scheme other than `snake_case` is destroyed on write.
- A hyphen in a UC name compiles: `` `sales-daily` `` works with backticks. It then requires backticks in every query, notebook, and tool that touches it, forever, and some tools quote incorrectly. Treat a hyphenated UC name as a defect even though the platform accepts it.
- Column names with spaces or other special characters require Delta column mapping and break downstream tools. Stay in `[a-z0-9_]`.
- Storage account and key vault names are globally unique across all of Azure. A name that passes review can still fail at deploy time; check availability in the infrastructure pipeline, not manually.
- The 24-character storage account budget is consumed by tokens. Adding a fourth token or lengthening `dataplat` breaks the scheme silently in the one resource type that cannot take hyphens.
- Environment embedded in a job or pipeline name means promotion renames it, and renames orphan run history and break references. Keep environment in the bundle target.
- Renaming a UC table does not update downstream views, dashboards, or GenAI retrieval configs. Fix a bad name immediately after creation or live with it; there is no cheap window later.

## Checklist

- [ ] Every UC object name matches `[a-z][a-z0-9_]*` and its pattern in the table above
- [ ] No layer prefix in any table or column name
- [ ] No environment token in any job, pipeline, or warehouse name
- [ ] Every Azure resource name is generated in infrastructure code from the standard tokens
- [ ] Storage account and key vault names fit the 24-character budget
- [ ] Every grant-bearing group name encodes role, scope, and environment
- [ ] Tag keys are `camelCase` and include the baseline set
- [ ] New name patterns are added to this doc before first use, not after

## Sources

- Databricks: [Names and identifiers](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-names)
- Microsoft Learn: [Naming rules and restrictions for Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules)
- Microsoft Learn (Cloud Adoption Framework): [Abbreviation recommendations for Azure resources](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
