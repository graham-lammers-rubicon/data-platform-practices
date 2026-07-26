# Genie and Metric Views

Defines how governed metrics are implemented (Unity Catalog metric views) and how natural-language access is served (Genie spaces on top of them). This is the implementation layer for the semantic rules in [Analytical dataset language](../practices/analytical-dataset-language.md) and [BI practices guidance](../practices/bi-practices-guidance.md).

## What this covers

- Metric views as the single implementation of governed metrics
- Genie spaces: scope, configuration as code, and quality loop
- Certification wiring and acceptance checks

## Metric views

A metric view is a Unity Catalog object whose definition is YAML: a source, dimensions, and measures with their aggregation expressions. Consumers query it with `MEASURE()` and group by any declared dimension; the aggregation rule travels with the definition, so re-aggregation is safe by construction. Generally available; native in AI/BI dashboards and Genie.

**Rules:**

- Every governed metric is defined in a metric view in the `gold` schema, pattern `<domain>_metrics` (`gold.sales_metrics`). One definition per metric name, per the [metric governance rules](../practices/analytical-dataset-language.md).
- Measures reference Silver measures or Gold serving columns, never another metric view.
- Ratio and rate metrics are declared as expressions over components (`SUM(shipped_quantity) / SUM(ordered_quantity)`), which is what makes them re-aggregate correctly. Never define a ratio measure over a pre-computed ratio column.
- Every dimension and measure carries a `comment`; the view carries owner and version the same way other Gold objects do.
- Metric views deploy from the domain repo like any other Gold object. The catalog comes from the deployment target; never hardcode an environment.
- Wide Gold serving tables (materialized views) remain the pattern for APIs, retrieval indexes, and exports. Metric views are the definition layer for BI, Genie, and dashboards.

**Pattern:**

```sql
CREATE OR REPLACE VIEW gold.sales_metrics
WITH METRICS
LANGUAGE YAML
AS $$
  version: 1.1
  source: gold.sales_product_monthly
  comment: "Sales KPIs. Owner: RevOps. v1 2025-01-01."
  dimensions:
    - name: Month
      expr: make_date(calendar_year, calendar_month, 1)
      comment: "Calendar month of sale"
    - name: Product
      expr: product_sku
      comment: "Product SKU"
  measures:
    - name: Revenue
      expr: SUM(revenue_usd)
      comment: "Booked line revenue, USD"
    - name: Fill Rate
      expr: SUM(shipped_quantity_num) / SUM(ordered_quantity_den)
      comment: "Shipped over ordered units. Safe to aggregate: components summed first."
$$
```

Query shape (all measures wrapped in `MEASURE()`, no `SELECT *`):

```sql
SELECT `Month`, MEASURE(`Revenue`) AS revenue, MEASURE(`Fill Rate`) AS fill_rate
FROM gold.sales_metrics
GROUP BY ALL ORDER BY ALL;
```

## Genie spaces

A Genie space is the natural-language interface to a domain's certified metrics. It translates questions to SQL against the tables and metric views it is scoped to.

**Rules:**

- One Genie space per domain, scoped to that domain's metric views and certified Gold serving tables. Bronze and Silver never appear in a space; this is the same boundary as the [access matrix](../governance/access-model.md).
- Build spaces on metric views first. A question Genie answers through a metric view is grounded in the governed definition; a question answered by ad hoc SQL over raw columns is a definition fork waiting to happen.
- Space configuration is code. Export the serialized space to JSON in the domain repo; changes go through PR and are pushed with the CLI (`databricks genie create-space` / `update-space`). A space edited only in the UI is a defect, same as any other click-ops artifact.
- Every space ships with sample questions, example question SQL, and text instructions. Example SQL must query the metric view, not re-derive the metric from base columns.
- Well-established recurring questions get [trusted assets](https://learn.microsoft.com/en-us/azure/databricks/genie/trusted-assets): example SQL queries or UC-registered functions that return a verified answer when they match the question. A trusted asset queries the metric view like all other example SQL.
- Text instructions carry the business context Genie cannot infer: which table answers which question class, thresholds, date-range conventions ("recent" means trailing 7 days), join keys.
- The space's warehouse is serverless and policy-governed ([Compute policies](compute-policies.md)).

**Quality loop:** treat Genie accuracy as testable, using the native [benchmarks feature](https://learn.microsoft.com/en-us/azure/databricks/genie/benchmarks): each space carries benchmark questions (up to 500) with a SQL answer, so accuracy is scored automatically by comparing result sets. Include two to four phrasings per question. Run benchmarks after each space change and after upstream schema changes; the Conversation API is for automating this in CI, not for hand-rolling scoring. Wrong or empty answers are fixed by adding trusted assets, example SQL, and instructions, not by telling users to rephrase.

## Certification wiring

- Only certified, decision-grade metric views back a Genie space presented for decision use. An uncertified metric may appear only in a space labeled informational ([BI practices guidance](../practices/bi-practices-guidance.md)).
- A metric definition change is a metric view change in the repo: version bump, documented boundary, benchmark questions re-run.
- Export events and definition forks remain the defect signals; a Genie space does not change the governance model, it inherits it.

## Sharp edges

- `SELECT *` does not work on a metric view, and every measure reference needs `MEASURE()`. Tools that generate bare SQL against it will fail; point them at a serving table instead.
- Joins belong in the metric view YAML, not in the consuming query. A query-time join to a metric view fails.
- Metric views require a current runtime (YAML 1.1 needs DBR 17.2+; some sub-features are newer). Pin warehouse and pipeline channels accordingly.
- Metric view materialization is experimental. Do not make it load-bearing; a wide Gold MV is the performance fallback.
- Genie serialized-space JSON is strict: every sample question, example SQL, and instruction needs a unique 32-hex `id`, and text fields are arrays. Export with the CLI and edit the exported file rather than hand-authoring.
- A Genie space over a mixed bag of raw tables produces confident wrong answers. Scope tightly, instruct explicitly, benchmark continuously.

## Checklist

- [ ] Every governed metric exists in exactly one metric view; no consumer re-implements the calculation
- [ ] Ratio measures are component expressions, not references to stored ratios
- [ ] Every dimension and measure has a comment; view carries owner and version
- [ ] Genie space scoped to metric views and certified Gold tables only
- [ ] Space JSON lives in the domain repo; UI state matches the repo
- [ ] Benchmark questions defined in the space with SQL answers; scores reviewed after every change
- [ ] Decision-grade spaces expose certified definitions only

## Sources

- Azure Databricks: [Metric views](https://learn.microsoft.com/en-us/azure/databricks/uc-semantics/metric-views/)
- Azure Databricks: [Create a metric view](https://learn.microsoft.com/en-us/azure/databricks/uc-semantics/metric-views/create)
- Azure Databricks: [AI/BI Genie](https://learn.microsoft.com/en-us/azure/databricks/genie/)
- Azure Databricks: [Genie benchmarks](https://learn.microsoft.com/en-us/azure/databricks/genie/benchmarks)
- Azure Databricks: [Genie trusted assets](https://learn.microsoft.com/en-us/azure/databricks/genie/trusted-assets)
- Databricks: [Unity Catalog Business Semantics GA](https://www.databricks.com/blog/redefining-semantics-data-layer-future-bi-and-ai)
