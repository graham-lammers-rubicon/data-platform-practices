# Databricks Genie Spaces

Defines how natural-language access to governed data is served: Genie spaces built on [metric views](databricks-metric-views.md) and certified Gold tables. A Genie space translates questions to SQL against the objects it is scoped to; the governance model it inherits comes from the [BI practices guidance](../practices/bi-practices-guidance.md) and the [access model](../governance/access-model.md).

## What this covers

- Space scope and configuration as code
- Trusted assets and instructions
- The quality loop: benchmarks
- Certification wiring

## Rules

- One Genie space per domain, scoped to that domain's metric views and certified Gold serving tables. Bronze and Silver never appear in a space; this is the same boundary as the access matrix.
- Build spaces on metric views first. A question Genie answers through a metric view is grounded in the governed definition; a question answered by ad hoc SQL over raw columns is a definition fork waiting to happen.
- Space configuration is code. Export the serialized space to JSON in the domain repo; changes go through PR and are pushed with the CLI (`databricks genie create-space` / `update-space`). A space edited only in the UI is a defect, same as any other click-ops artifact.
- Every space ships with sample questions, example question SQL, and text instructions. Example SQL must query the metric view, not re-derive the metric from base columns.
- Well-established recurring questions get [trusted assets](https://learn.microsoft.com/en-us/azure/databricks/genie/trusted-assets): example SQL queries or UC-registered functions that return a verified answer when they match the question. A trusted asset queries the metric view like all other example SQL.
- Text instructions carry the business context Genie cannot infer: which table answers which question class, thresholds, date-range conventions ("recent" means trailing 7 days), join keys.
- The space's warehouse is serverless and policy-governed ([Compute policies](databricks-compute-policies.md)).

## Quality loop

Treat Genie accuracy as testable, using the native [benchmarks feature](https://learn.microsoft.com/en-us/azure/databricks/genie/benchmarks): each space carries benchmark questions (up to 500) with a SQL answer, so accuracy is scored automatically by comparing result sets. Include two to four phrasings per question. Run benchmarks after each space change and after upstream schema changes; the Conversation API is for automating this in CI, not for hand-rolling scoring. Wrong or empty answers are fixed by adding trusted assets, example SQL, and instructions, not by telling users to rephrase.

## Certification wiring

- Only certified, decision-grade metric views back a Genie space presented for decision use. An uncertified metric may appear only in a space labeled informational ([BI practices guidance](../practices/bi-practices-guidance.md)).
- A metric definition change ([Metric views](databricks-metric-views.md)) re-runs the consuming spaces' benchmarks.
- Export events and definition forks remain the defect signals; a Genie space does not change the governance model, it inherits it.

## Sharp edges

- Genie serialized-space JSON is strict: every sample question, example SQL, and instruction needs a unique 32-hex `id`, and text fields are arrays. Export with the CLI and edit the exported file rather than hand-authoring.
- A Genie space over a mixed bag of raw tables produces confident wrong answers. Scope tightly, instruct explicitly, benchmark continuously.
- Benchmark questions without a SQL answer cannot be auto-scored; they queue for manual review. Give every benchmark question a SQL answer.

## Checklist

- [ ] Genie space scoped to metric views and certified Gold tables only
- [ ] Space JSON lives in the domain repo; UI state matches the repo
- [ ] Recurring questions covered by trusted assets querying the metric view
- [ ] Benchmark questions defined in the space with SQL answers; scores reviewed after every change
- [ ] Decision-grade spaces expose certified definitions only

## Sources

- Azure Databricks: [AI/BI Genie](https://learn.microsoft.com/en-us/azure/databricks/genie/)
- Azure Databricks: [Genie benchmarks](https://learn.microsoft.com/en-us/azure/databricks/genie/benchmarks)
- Azure Databricks: [Genie trusted assets](https://learn.microsoft.com/en-us/azure/databricks/genie/trusted-assets)
