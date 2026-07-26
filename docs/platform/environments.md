# Environments

Defines the environment model: two workspace tiers, environment-per-catalog, one-way cross-tier access, and bundle-only promotion.

## What this covers

- Workspace and catalog topology
- Deployment and immutability rules
- Cross-tier data access
- Cost and test data rules

## Topology

Two workspace tiers. Environments are catalogs and bundle targets, not workspaces.

| Tier | Workspace | Environments (catalogs) |
| --- | --- | --- |
| Prod | `dbw-dataplat-prd-eus2-001` | `prd_catalog` |
| Nonprod | `dbw-dataplat-np-eus2-001` | `dev_catalog`, `qa_catalog`, `test_catalog` |

All catalogs share the regional Unity Catalog metastore. Workspace-catalog bindings enforce the tier boundary (below). Tokens: [Naming conventions](naming-conventions.md).

## Deployment rules

- Every deployable asset in every environment comes from a bundle (DAB). Bundle targets: `dev`, `qa`, `test`, `prd`.
- The `dev` target MAY use `mode: development` (user-prefixed resources, paused triggers, development-mode pipelines). `qa`, `test`, and `prd` targets MUST use `mode: production`, deployed by CI as the deployment service principal.
- Click-ops is allowed only in user home folders in the nonprod workspace: notebooks, queries, draft dashboards. Anything that MAY be promoted MUST live in a repo and deploy via a bundle. There is no promotion path for click-ops artifacts.
- Deployed assets are immutable: changed only by redeploying from source. Editing a deployed job or pipeline in place is a defect.
- Infrastructure is immutable in every tier: Terraform and bundles only. This includes nonprod.
- Promotion is redeployment of the same commit to the next target: `dev` → `qa` → `test` → `prd`. `prd` deploys only from `main`.

## Cross-tier access

- `prd_catalog`: isolation mode `ISOLATED`; bound read-write to the prod workspace, bound `BINDING_TYPE_READ_ONLY` to the nonprod workspace. Nonprod reads prod for audit and reconciliation; the binding blocks all writes.
- Nonprod catalogs: bound to the nonprod workspace only. Prod has no path to nonprod.
- The read-only binding is workspace-level enforcement, not a grant. UC privileges still decide who reads prod data from nonprod (see [Access model](../governance/access-model.md)).

## Cost

- Nonprod jobs and pipelines SHOULD run on manual or CI trigger, not schedules. Development mode pauses triggers by default; `qa` and `test` targets set the trigger pause preset explicitly. Unpausing a nonprod schedule requires a stated reason.
- Continuous-mode pipelines are prod-only.
- Sizing and termination: [Compute policies](compute-policies.md).

## Test data

Snapshots are provisioned, never hand-loaded:

- Delta tables: `DEEP CLONE` from `prd_catalog` (readable via the binding) into the target nonprod catalog.
- Lakebase (Postgres): copy-on-write branches from the parent branch, TTL-expiring (max 30 days) or reset-to-parent.
- Streaming tables and materialized views do not support `CLONE`. Snapshot their upstream Delta sources and rebuild, or materialize to a plain Delta table first.
- Clones are point-in-time; refresh means re-clone.

## Sharp edges

- Default catalog isolation is `OPEN`: an unbound `prd_catalog` is reachable read-write from every workspace in the metastore. Set `ISOLATED` before the first prod data lands.
- The read-only binding blocks writes, not reads: prod PII is visible from nonprod to anyone holding UC privileges. Grant prod read access from nonprod per role, never broadly.
- Development-mode prefixes (`[dev <user>]`) intentionally break naming conventions in dev. Never hardcode resource names downstream of a dev deploy.
- A `qa`/`test` target left on `mode: development` gets user-prefixed names and paused-forever triggers: promotion tests pass in a shape prod never has.
- Deep clones duplicate storage and drift immediately; a paused schedule still costs compute every triggered run.
- Lakebase on Azure is Beta (eastus2 available). Validate before making branches load-bearing in the test strategy.

## Checklist

- [ ] `prd_catalog` is `ISOLATED`, read-write to prod only, read-only to nonprod
- [ ] Nonprod catalogs are bound to the nonprod workspace only
- [ ] Every `qa`/`test`/`prd` deployment runs in CI as the deployment SP with `mode: production`
- [ ] Every promoted asset traces to a repo commit and bundle target
- [ ] No unpaused schedule in nonprod without a documented reason
- [ ] Test datasets are provisioned via deep clone or Lakebase branch, refresh procedure documented
- [ ] No manual edits to deployed assets or infrastructure in any tier

## Sources

- Databricks: [Deployment modes](https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes)
- Databricks: [Limit catalog access to specific workspaces](https://docs.databricks.com/aws/en/catalogs/binding)
- Databricks: [Clone a table](https://docs.databricks.com/aws/en/delta/clone)
- Azure Databricks: [Lakebase branches](https://learn.microsoft.com/en-us/azure/databricks/oltp/projects/branches)
