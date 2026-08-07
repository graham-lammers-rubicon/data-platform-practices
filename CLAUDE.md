# CLAUDE.md

Guidance for working in this repository. This repo captures data platform practices for Databricks-based teams: onboarding policies, medallion architecture, data product standards, and dataset design conventions. It is a documentation repo. Output here is normative guidance that other engineers execute against, so precision matters more than volume.

This repo is written for humans and agents. Agents such as Claude Code and Databricks Genie SHOULD use it as the baseline guidance when building out a Data Intelligence Platform on Databricks hosted in Azure: follow the rules in these docs before inventing patterns, and cite the doc being applied.

## Repo layout

```text
docs/
  index.md                       Site entry point with role router
  onboarding/                    Role-based entry paths
    data-engineer.md
    bi-analyst.md
    platform-engineer.md
  practices/                     Design and delivery standards (reference docs for the rules below)
    spec-driven-development.md   Spec contents, workflow, acceptance in CI
    data-products.md             Data product definition, principles, anatomy
    pipeline-automation.md       Automation mantra, human roles, four pillars
    medallion-data-practices.md
    analytical-dataset-language.md
    bi-practices-guidance.md
    tidy-data.md
  platform/                      Infrastructure reference, prefixed by system where one owns the doc
    azure-infrastructure.md      Hybrid compute (VNet-injected + serverless via NCC), hub-and-spoke, private endpoints, UC wiring, storage layout, Terraform
    databricks-compute-policies.md      Workload classes, policy set, cost attribution
    databricks-environments.md   Two tiers (prod/np), env-per-catalog, DAB-only promotion
    decision-register.md         Dated architecture decisions and open items with owners
    databricks-genie-spaces.md   Genie spaces as code: scope, trusted assets, benchmarks
    databricks-metric-views.md   Metric views as the governed metric implementation
    databricks-service-principal-auth.md  SP types, standard identities, auth ranking, lifecycle
    github-cicd-and-deployment.md  GitHub Actions, promotion gates, OIDC identity
    naming-conventions.md        Case styles, name patterns, standard tokens (cross-system)
    metadata-and-comments.md     UC COMMENTs, commit messages, docstrings (cross-system)
    secrets-and-credentials.md   Storage hierarchy, KV-backed scopes, repo hygiene (cross-system)
    resilience.md                Stub: DR/BCP decision register (cross-system)
  governance/                    Access and usage policy
    access-model.md              Authoritative home of the access matrix
    responsible-use.md           Data handling, compute accountability, GenAI rules
    data-lifecycle.md            Stub: classification, retention, deletion decision register
```

A separate root, `build-out/`, holds Workstream 04 build-phase working material: Nimble Gravity's vendor-authored WS04 plans (drafts under joint review; client revisions land by PR and are fed back to NG) and the vendor-driven work items. Nothing under `build-out/` is normative; where it conflicts with `docs/`, `docs/` wins, and conflicts are recorded in `docs/platform/decision-register.md`. Never use an edit to a vendor document to record a decision.

Docs marked stub carry a `> **Status: stub.**` notice: scope is agreed, rules are unwritten, not normative. When writing a stub's content, remove the notice.

The sections below are the distilled rules. The full references with patterns, examples, and checklists live in `docs/practices/`. When the summary here and a reference doc disagree, the reference doc wins; fix the summary.

When adding a page, register it in the parent `index.md`. Keep the doc tree and the index files in sync.

## Writing standards for this repo

- Use normative language deliberately: "must" is a requirement, "should" is a strong recommendation, "may" is an option.
- Every practice doc states what it covers, the rules, the sharp edges (failure modes), and a checklist or acceptance criterion. A rule without a way to verify compliance is incomplete.
- Cite primary sources (official Databricks docs, standards, papers) at the bottom of each doc. Do not assert platform behavior from memory; verify against docs or the relevant skill first.
- Plain language, short sentences, no filler. No em dashes.
- Examples use SQL or Lakeflow Spark Declarative Pipelines syntax consistent with current Databricks documentation. `LIVE.` and `APPLY CHANGES INTO` are legacy; use `CREATE FLOW ... AS AUTO CDC INTO`. Pipeline code uses two-part `schema.table` names; the catalog comes from the bundle target, never hardcoded.
- Cite the Azure version of Databricks docs (`learn.microsoft.com/en-us/azure/databricks/...`), never `docs.databricks.com/aws/...`.

---

## 1. Onboarding, Platform, and Governance

Onboarding content lives under `docs/onboarding/` as role-based paths (data engineer, BI analyst, platform engineer): reading order, first-week checklist, and the rules that role is held to. Infrastructure reference docs live under `docs/platform/` (Azure footprint, environments, CI/CD, compute, naming, secrets, service principal auth). Access and usage policy lives under `docs/governance/`; the access matrix in section 2 below is authoritatively defined in `docs/governance/access-model.md`.

Rules that apply across these docs:

- **Spec before build, as guidance.** Data products and platform changes should start from a spec: requirements, data contracts, acceptance criteria, validation plan. Not an absolute gate, but nothing is certified without its contract recorded. See `docs/practices/spec-driven-development.md`.
- **Least privilege by default.** Access is requested per role and per layer, not granted broadly. Downstream consumers never get Bronze or Silver access (see medallion rules below).
- **Everything is named by convention.** Catalogs, schemas, jobs, and pipelines follow the naming conventions doc. No ad hoc names in examples or templates.
- **Everything is configuration-as-code.** Workspace, catalog, schema, and permission configuration lives in repos and deploys via Terraform or bundles. A UI-made change to any of these is a defect; the repo is the change record.
- **No secrets in code or docs.** Credentials go through secret scopes or service principal auth. Docs must never contain real hostnames, tokens, or workspace IDs.
- **Cost is governed.** Compute examples in docs must reflect the compute policies: right-sized SKUs, auto-termination, tagging for attribution.

---

## 2. Medallion Data Architecture

Bronze, Silver, Gold is a structural promise between producers and consumers, not a naming convention. Every layer lives in Unity Catalog. Lineage and audit come from UC system tables, not spreadsheets.

```text
<env>_catalog
  ├── bronze    →  <entity>              (streaming tables)
  ├── silver    →  <domain>              (fact tables)
  └── gold      →  <domain>_<grain>      (materialized views or Delta)
```

No layer prefixes in table or column names. The schema tells you the layer; the COMMENT tells you what it is.

### Bronze: capture everything, transform nothing

- Raw, append-only, schema-on-read. One table per source entity. No joins, casting, renaming, or business logic.
- Always configure `rescuedDataColumn => '_rescued_data'`. Add system columns on ingest: `_ingest_timestamp`, `_source_file`, `_rescued_data`.
- Wide or messy source shapes land as-is. Restructuring is Silver's job.
- CDC feeds land as append-only event tables, operations and sequence columns intact. No changes are applied in Bronze; the event log is the replay source for all downstream state.

### Silver: validate, clean, conform

- Cast types, deduplicate, handle nulls, apply conformed keys. Declare quality expectations with EXPECT constraints (DROP ROW or FAIL UPDATE). Never write to Silver directly from source.
- **SCD Type 2 is derived in Silver** via `CREATE FLOW ... AS AUTO CDC INTO ... STORED AS SCD TYPE 2`, reading the Bronze event table. It is rebuildable by replay; a backfill means full-refreshing the Silver target, never re-feeding events into an existing one.
- Silver declares its grain in the table COMMENT; that is mandatory. Tidy shape (long and atomic: one variable per column, one observation per row, wide sources unpivoted, compound codes split into typed columns) is an optional, low-priority practice, not a gate. See section 4.
- Store raw measures at source grain. No business metric calculations.
- Semi-additive measures (balances, headcount, inventory) are labeled in the column COMMENT. Non-additive ratios are not stored; store numerator and denominator components separately.

### Gold: govern, enrich, serve

- Gold is the governed semantic layer and the only layer consuming services touch: analytics, GenAI retrieval, APIs, MCP servers. If any consumer has a connection to Bronze or Silver, that is a defect.
- Every Gold object has an owner and version (`TBLPROPERTIES` or catalog tags). Definitions reference Silver measures; a metric view may source a Gold serving table. Nothing references another derived Gold definition.
- Gold is wide by design, aggregated to a declared grain, and must pass the pivot test (entity rows × period columns × one additive measure).
- Non-additive outputs expose numerator and denominator as separate columns and are documented as "do not re-aggregate."
- Cross-domain joins (e.g., CAC across marketing, HR, and sales) are defined once in Gold, never re-derived per consumer.

### Access matrix

Authoritative version, with the environment axis and job identities: `docs/governance/access-model.md`. Summary:

| Role | Bronze | Silver | Gold |
| --- | --- | --- | --- |
| Domain pipeline SP | READ/WRITE (own domain) | READ/WRITE (own domain) | READ/WRITE (own domain) |
| Data engineers | dev: READ/WRITE; else READ | dev: READ/WRITE; else READ | dev: READ/WRITE; else READ |
| Analysts / data scientists | none | READ (approved) | READ |
| Consuming services | none | none | READ |

Human WRITE exists only in `dev_catalog`. Write isolation between domains sharing a schema comes from table ownership.

### Pipeline architecture

One Lakeflow SDP pipeline per domain, owning Bronze through Gold for that scope. Conformed dimensions are built by the platform-owned `conformed-dimensions` pipeline. Cross-domain Gold joins run in a separate pipeline or job, as their own service principal, after upstream Silver completes. Writes outside the pipeline bypass expectations and the declared contract; route all writes through the domain pipeline.

---

## 3. Data Products

A data product is a governed dataset published to the analytical layer with a declared contract. The contract has five elements: period, grain, dimensions, measures, metrics. Principles, anatomy, and consumption patterns: `docs/practices/data-products.md`; the automation standard those products are built under: `docs/practices/pipeline-automation.md`.

### Grain is the contract

- Declare the grain in plain English before any schema work: "One row per order line, per calendar day." If the sentence is ambiguous, the design is not ready.
- Grain type must be identified: transaction, periodic snapshot, or accumulating snapshot. The type determines aggregation behavior across time.
- Never mix grains in one fact table. Two measures implying different grains belong in different tables. Grain mixing is the top source of silent double-counting.
- Store atomic grain. Roll-up is cheap; roll-down is impossible.
- Document the grain as a table COMMENT so it cannot be missed.
- Accumulating snapshots are updated, not appended. Pipelines that treat them as append-only will duplicate or miss milestone data.

### Period and dimensions

- Every fact table attaches to a conformed period dimension with declared grain and hierarchy, carrying both calendar and fiscal attributes when the business uses a fiscal calendar.
- Core conformed dimensions: period, entity, product, location, channel, org. Same surrogate keys and shared attributes across every domain. Conformed does not mean identical; domain-specific attributes may extend a dimension, but shared keys and attributes must never diverge.
- Dimensions are stored flat (denormalized) with a declared SCD strategy.

### Measures vs. metrics

This distinction is where semantic layers break down. Do not conflate them.

- **Measure:** a raw stored fact in a fact table (Silver). Additive, semi-additive, or non-additive.
- **Metric:** a governed business calculation defined once in the semantic layer (Gold), with a name, plain-English definition, aggregation rule, filters, time intelligence, grain, owner, and version.
- You store measures. You define metrics. You report metrics to the business.
- One definition per metric name. Metrics reference measures, not other metrics. Definitions are versioned; when one changes, the version boundary is documented.
- Non-additive metrics declare how they aggregate across dimensions (total conversions / total clicks, not the average of channel rates).
- Metrics are implemented as Unity Catalog metric views (`gold.<domain>_metrics`), consumed by dashboards and Genie from the same definition. See `docs/platform/databricks-metric-views.md` and `docs/platform/databricks-genie-spaces.md`.

### Decision-first design (BI standard)

- Dashboards are informational by default; the platform exists to drive insight, action, and defensible decisions. Every published output is classified informational or decision-grade. Decision-grade outputs name the decision, decision-maker, and follow-on action; only they may be certified.
- The primary measure of the platform is whether it supports better, faster, more accountable decisions. Dashboard counts and refresh speed are operational indicators only. The platform team instruments its own decision-support performance: certified-output usage, export events as a defect signal, request-to-certified time.
- Reuse before build: check the semantic layer for an existing certified definition before authoring a new metric or entity. Extend, don't fork; a new name requires genuinely new meaning.
- Design backward from the decision: what decision, how often, what information, how current, what accuracy, what action follows.
- These behaviors are defects, regardless of infrastructure quality: exporting to spreadsheets to get numbers, manual reconciliation before meetings, teams disagreeing on shared KPI definitions.
- A metric with unresolved definitional questions (entity resolution, disputed records, currency, governing timestamp) must not be certified for decision use.
- Executive metrics carry elevated controls: reproducible, traceable, access-controlled, no manual spreadsheet steps.
- Real-time delivery requires a documented latency assessment. High refresh frequency on unreliable data is false precision.
- Self-service means exploring certified Gold data. Direct business-user access to raw tables is not self-service; it is distributed data engineering without standards.

### Acceptance criterion: the pivot test

A published dataset must produce entity rows × period columns × one additive measure with a standard GROUP BY or PIVOT. If it fails, the measure is non-additive and needs pre-derivation, the grain is mixed, or the period is not conformed. Fix the structure before building on top of it.

---

## 4. Tidy Data (Optional, Low Priority)

Reference: Wickham, "Tidy Data," Journal of Statistical Software 59(10), 2014.

Status: optional practice, nice-to-have. The Silver-long / Gold-wide reshaping is a hard transition for people and tooling; adopt it per domain when capacity allows, and do not block delivery on it. Mandatory regardless of stored shape: grain declared in the COMMENT, typed columns, additivity labeled, and the pivot test on published datasets.

Three rules:

1. Each variable is a column.
2. Each observation is a row.
3. Each type of observational unit is its own table.

Target shapes by layer, where adopted:

| Layer | Shape | Rationale |
| --- | --- | --- |
| Bronze | Whatever the source sends, including messy and wide | Source fidelity; evidence for reprocessing |
| Silver | Tidy: long, atomic, one variable per column | Analysis, aggregation in any direction, forecasting-ready |
| Gold | Intentionally wide at a declared grain | Consumption; the pivot test is the acceptance check |

Messy patterns and their fixes, applied in Silver where the practice is adopted:

- Column headers holding values (jan_revenue, feb_revenue): unpivot to long.
- Multiple variables in one column ("US-ENT-0042"): split into typed columns.
- Variables in both rows and columns: pivot longer, then wider.
- Mixed observational units in one table: normalize into separate tables with keys.
- One unit spread across many files: union with a discriminator column.

Sharp edges: tidy is an analysis standard, not a storage mandate; benchmark before assuming long is faster. Forecasting libraries expect long format (entity, period, value) and do not know a measure is semi-additive; feed them the correctly aggregated series or forecasts are wrong by construction.

---

## Sharp edges summary (cross-cutting)

- Grain mixing is silent and lethal. Declare once, enforce strictly.
- Semi-additive measures SUMmed across time produce wrong numbers. Label them and enforce in the metric definition.
- Stored ratios lie when re-aggregated. Store components; compute at the declared grain.
- Definitions living in consuming services diverge. One definition in Gold; revoke lower-layer access.
- Skipping `rescuedDataColumn` silently drops unknown fields; the discovery comes months later.
- Period without a fiscal calendar declaration causes silent misalignment when fiscal and calendar attributes are used interchangeably.
