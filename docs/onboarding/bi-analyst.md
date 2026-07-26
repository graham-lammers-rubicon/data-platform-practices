# Onboarding: BI Analyst

You turn governed Gold data into outputs that drive decisions. This page is your reading order and first-week checklist.

## Read in this order

1. [Access model](../governance/access-model.md) - you work in Gold; Silver access is by approval, Bronze is off limits
2. [BI practices guidance](../practices/bi-practices-guidance.md) - the core reference: decision-first design, certification, self-service boundaries
3. [Genie and metric views](../practices/genie-and-metric-views.md) - how metrics are implemented and how you query them
4. [Analytical dataset language](../practices/analytical-dataset-language.md) - period, grain, dimensions, measures, metrics; the vocabulary every dataset contract uses
5. [Tidy data](../practices/tidy-data.md) - why Gold is wide, and the pivot test as the acceptance check
6. [Medallion data practices](../practices/medallion-data-practices.md) - skim for context on where your data comes from and why layer boundaries exist

## First-week checklist

- [ ] Workspace and Gold access granted per the [access model](../governance/access-model.md)
- [ ] Locate the domain metric views (`gold.<domain>_metrics`); pick three metrics and trace each to its Silver measures via the YAML definition (`SHOW CREATE TABLE`)
- [ ] Query a metric view: measures wrapped in `MEASURE()`, grouped by its declared dimensions (`SELECT *` does not work on metric views)
- [ ] Ask the domain Genie space one of its sample questions and confirm the generated SQL hits the metric view, not raw tables
- [ ] Classify one existing dashboard as informational or decision-grade and check it against the decision-first criteria
- [ ] Run the pivot test on one published dataset

## Rules you will be held to

- Reuse before build: check the domain metric view for an existing certified definition before authoring a new metric. Extend the metric view through its owner; never fork the calculation into a dashboard.
- You report metrics, not measures. One definition per metric name, defined once in a metric view in Gold, versioned. Dashboards and Genie consume that definition; re-deriving a metric from base columns in a BI tool is a defect.
- Exporting to spreadsheets to get numbers, manual reconciliation before meetings, and forked KPI definitions are defects, regardless of infrastructure quality.
- A metric with unresolved definitional questions must not be certified for decision use.
- Non-additive metrics declare how they aggregate. Never average ratios across dimensions.
