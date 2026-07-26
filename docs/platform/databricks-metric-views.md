# Databricks Metric Views

Defines how governed metrics are implemented: Unity Catalog metric views. This is the implementation layer for the semantic rules in [Analytical dataset language](../practices/analytical-dataset-language.md) and [BI practices guidance](../practices/bi-practices-guidance.md). Natural-language access on top of metric views: [Databricks Genie spaces](databricks-genie-spaces.md).

## What this covers

- Metric views as the single implementation of governed metrics
- Definition, deployment, and query rules
- Certification and versioning

## Rules

A metric view is a Unity Catalog object whose definition is YAML: a source, dimensions, and measures with their aggregation expressions. Consumers query it with `MEASURE()` and group by any declared dimension; the aggregation rule travels with the definition, so re-aggregation is safe by construction. Generally available; native in AI/BI dashboards and Genie.

- Every governed metric is defined in a metric view in the `gold` schema, pattern `<domain>_metrics` (`gold.sales_metrics`). One definition per metric name, per the [metric governance rules](../practices/analytical-dataset-language.md).
- Measures reference Silver measures or Gold serving columns, never another metric view.
- Ratio and rate metrics are declared as expressions over components (`SUM(shipped_quantity) / SUM(ordered_quantity)`), which is what makes them re-aggregate correctly. Never define a ratio measure over a pre-computed ratio column.
- Every dimension and measure carries a `comment`; the view carries owner and version the same way other Gold objects do.
- Metric views deploy from the domain repo like any other Gold object. The catalog comes from the deployment target; never hardcode an environment.
- Wide Gold serving tables (materialized views) remain the pattern for APIs, retrieval indexes, and exports. Metric views are the definition layer for BI, dashboards, and [Genie spaces](databricks-genie-spaces.md), which build on metric views, not raw tables.
- A metric definition change is a metric view change in the repo: version bump, documented boundary ([BI practices guidance](../practices/bi-practices-guidance.md)), and the consuming Genie spaces' benchmarks re-run.

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

## Sharp edges

- `SELECT *` does not work on a metric view, and every measure reference needs `MEASURE()`. Tools that generate bare SQL against it will fail; point them at a serving table instead.
- Joins belong in the metric view YAML, not in the consuming query. A query-time join to a metric view fails.
- Metric views require a current runtime (YAML 1.1 needs DBR 17.2+; some sub-features are newer). Pin warehouse and pipeline channels accordingly.
- Metric view materialization is experimental. Do not make it load-bearing; a wide Gold MV is the performance fallback.

## Checklist

- [ ] Every governed metric exists in exactly one metric view; no consumer re-implements the calculation
- [ ] Ratio measures are component expressions, not references to stored ratios
- [ ] Every dimension and measure has a comment; view carries owner and version
- [ ] Metric views deploy from the repo; no hardcoded catalog
- [ ] Definition changes carry a version bump and documented boundary

## Sources

- Azure Databricks: [Metric views](https://learn.microsoft.com/en-us/azure/databricks/uc-semantics/metric-views/)
- Azure Databricks: [Create a metric view](https://learn.microsoft.com/en-us/azure/databricks/uc-semantics/metric-views/create)
- Databricks: [Unity Catalog Business Semantics GA](https://www.databricks.com/blog/redefining-semantics-data-layer-future-bi-and-ai)
