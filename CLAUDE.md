# CLAUDE.md

Guidance for working in this repository. This repo captures data platform practices for Databricks-based teams: onboarding policies, medallion architecture, data product standards, and dataset design conventions. It is a documentation repo. Output here is normative guidance that other engineers execute against, so precision matters more than volume.

## Repo layout

```text
docs/
  index.md                       Site entry point
  onboarding/                    Team onboarding and governance
    cloud-resources/             Databricks-specific operational policies
  practices/                     Full reference docs for the rules below
    medallion-data-practices.md
    analytical-dataset-language.md
    bi-practices-guidance.md
    tidy-data.md
```

The sections below are the distilled rules. The full references with patterns, examples, and checklists live in `docs/practices/`. When the summary here and a reference doc disagree, the reference doc wins; fix the summary.

When adding a page, register it in the parent `index.md`. Keep the doc tree and the index files in sync.

## Writing standards for this repo

- Use normative language deliberately: "must" is a requirement, "should" is a strong recommendation, "may" is an option.
- Every practice doc states what it covers, the rules, the sharp edges (failure modes), and a checklist or acceptance criterion. A rule without a way to verify compliance is incomplete.
- Cite primary sources (official Databricks docs, standards, papers) at the bottom of each doc. Do not assert platform behavior from memory; verify against docs or the relevant skill first.
- Plain language, short sentences, no filler. No em dashes.
- Examples use SQL or Lakeflow Spark Declarative Pipelines syntax consistent with current Databricks documentation. `LIVE.` syntax is legacy; use fully qualified `catalog.schema.table` names for cross-pipeline reads.

---

## 1. Onboarding Policies

Onboarding content lives under `docs/onboarding/`. Cloud resource policies live under `docs/onboarding/cloud-resources/` and cover: access requests, compute policies, environment guide, naming conventions, secrets and credentials, service principal auth for Databricks Apps, responsible use, and spec-driven development.

Rules that apply across onboarding docs:

- **Spec before build.** Platform work starts from a spec: requirements, data contracts, acceptance criteria, and validation plan. See `docs/onboarding/cloud-resources/spec-driven-development.md`.
- **Least privilege by default.** Access is requested per role and per layer, not granted broadly. Downstream consumers never get Bronze or Silver access (see medallion rules below).
- **Everything is named by convention.** Catalogs, schemas, jobs, and pipelines follow the naming conventions doc. No ad hoc names in examples or templates.
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
- Always configure `rescuedDataColumn`. Add system columns on ingest: `_ingest_timestamp`, `_source_file`, `_pipeline_id`, `_is_quarantined`, `_raw_payload`.
- Wide or messy source shapes land as-is. Restructuring is Silver's job.
- **SCD Type 2 belongs in Bronze** in this practice, via Lakeflow AUTO CDC with `STORED AS SCD TYPE 2`. A dimension change arriving from source is a source event; if Bronze drops the prior version, history is unrecoverable. Silver reads the versioned entity and does not reconstruct history.

### Silver: validate, clean, conform

- Cast types, deduplicate, handle nulls, apply conformed keys. Declare quality expectations with EXPECT constraints (DROP ROW or FAIL UPDATE). Never write to Silver directly from source.
- Silver is long and atomic: one variable per column, one observation per row, grain declared in the table COMMENT. Unpivot wide source shapes here. Split compound codes into typed columns.
- Store raw measures at source grain. No business metric calculations.
- Semi-additive measures (balances, headcount, inventory) are labeled in the column COMMENT. Non-additive ratios are not stored; store numerator and denominator components separately.

### Gold: govern, enrich, serve

- Gold is the governed semantic layer and the only layer consuming services touch: analytics, GenAI retrieval, APIs, MCP servers. If any consumer has a connection to Bronze or Silver, that is a defect.
- Every Gold object has an owner and version (`TBLPROPERTIES` or catalog tags). Definitions reference Silver measures, not other Gold objects.
- Gold is wide by design, aggregated to a declared grain, and must pass the pivot test (entity rows × period columns × one additive measure).
- Non-additive outputs expose numerator and denominator as separate columns and are documented as "do not re-aggregate."
- Cross-domain joins (e.g., CAC across marketing, HR, and sales) are defined once in Gold, never re-derived per consumer.

### Access matrix

| Role | Bronze | Silver | Gold |
|---|---|---|---|
| Pipeline service principal | WRITE | WRITE | WRITE |
| Data engineers | READ | READ/WRITE | READ |
| Analysts / data scientists | none | READ (approved) | READ |
| Consuming services | none | none | READ |

### Pipeline architecture

One Lakeflow SDP pipeline per domain, owning Bronze through Gold for that scope. Cross-domain Gold joins run in a separate pipeline or job after upstream Silver completes. All writes route through UC managed tables; writes outside Lakeflow create lineage gaps.

---

## 3. Data Products

A data product is a governed dataset published to the analytical layer with a declared contract. The contract has five elements: period, grain, dimensions, measures, metrics.

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

### Decision-first design (BI standard)

- The primary measure of the platform is whether it supports better, faster, more accountable decisions. Dashboard counts and refresh speed are operational indicators only.
- Design backward from the decision: what decision, how often, what information, how current, what accuracy, what action follows.
- These behaviors are defects, regardless of infrastructure quality: exporting to spreadsheets to get numbers, manual reconciliation before meetings, teams disagreeing on shared KPI definitions.
- A metric with unresolved definitional questions (entity resolution, disputed records, currency, governing timestamp) must not be certified for decision use.
- Executive metrics carry elevated controls: reproducible, traceable, access-controlled, no manual spreadsheet steps.
- Real-time delivery requires a documented latency assessment. High refresh frequency on unreliable data is false precision.
- Self-service means exploring certified Gold data. Direct business-user access to raw tables is not self-service; it is distributed data engineering without standards.

### Acceptance criterion: the pivot test

A published dataset must produce entity rows × period columns × one additive measure with a standard GROUP BY or PIVOT. If it fails, the measure is non-additive and needs pre-derivation, the grain is mixed, or the period is not conformed. Fix the structure before building on top of it.

---

## 4. Tidy Data Recommendations

Reference: Wickham, "Tidy Data," Journal of Statistical Software 59(10), 2014.

Three rules:

1. Each variable is a column.
2. Each observation is a row.
3. Each type of observational unit is its own table.

How this maps to the medallion layers:

| Layer | Shape | Rationale |
|---|---|---|
| Bronze | Whatever the source sends, including messy and wide | Source fidelity; evidence for reprocessing |
| Silver | Tidy: long, atomic, one variable per column | Analysis, aggregation in any direction, forecasting-ready |
| Gold | Intentionally wide at a declared grain | Consumption; the pivot test is the acceptance check |

Messy patterns and their fixes, applied in Silver:

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
