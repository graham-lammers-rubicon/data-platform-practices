# Practices

Design and delivery standards. These are the authoritative versions of the rules summarized in the repo-level CLAUDE.md; when they disagree, these docs win.

The docs form one arc: shape the data, deliver the work, serve decisions. Read them in that order the first time; after that, each stands alone.

## 1. Shape the data

- [Analytical dataset language](analytical-dataset-language.md) - the contract vocabulary: period, grain, dimensions, measures, metrics. Start here; every other doc uses these terms.
- [Medallion data practices](medallion-data-practices.md) - applies the contract to Bronze/Silver/Gold: layer rules, Lakeflow SDP patterns, SCD2 placement.
- [Data products](data-products.md) - the packaging standard: what turns a governed dataset into a product with an owner, contract, and consumers.
- [Tidy data](tidy-data.md) - the theory behind the shapes: why Silver is long and Gold is wide. Optional and low priority; background reading, not rules.

## 2. Deliver the work

- [Spec-driven development](spec-driven-development.md) - contract before code: what a spec contains and how acceptance runs in CI.
- [Pipeline automation](pipeline-automation.md) - the delivery philosophy: machines run the pipelines, humans shape the system; the four pillars every solution must satisfy.

## 3. Serve decisions

- [BI practices guidance](bi-practices-guidance.md) - decision-first BI: certification, the semantic layer, latency, self-service, executive controls.

Implementation references for this arc live in [Platform](../platform/index.md): metric views and Genie spaces implement the semantic layer, environments and CI/CD implement delivery. Per-role reading orders: [Onboarding](../onboarding/index.md).
