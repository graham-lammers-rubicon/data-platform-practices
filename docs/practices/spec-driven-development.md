# Spec-driven Development

Platform work starts from a spec, not from code. A spec states what is being built, for whom, the contract it honors, and how everyone will know it works, before implementation begins. The spec is the source CI reads acceptance criteria from ([CI/CD](../platform/cicd-and-deployment.md)); a deploy without a validation step is incomplete.

This repo is written for humans and agents. A spec is the primary instruction an agent receives for build work: agents follow the spec, cite it, and stop when the spec is ambiguous rather than inventing intent.

## What this covers

- What a spec contains and where it lives
- The workflow from spec to certified output
- Rules and failure modes

## What a spec contains

Every spec has five sections. A section that does not apply says so explicitly; it is never silently absent.

1. **Decision or workflow served.** What decision, made by whom, how often, and what action follows. Decision-first design comes from [BI practices guidance](bi-practices-guidance.md); work that serves no named decision is informational and says so.
2. **Requirements.** Sources, refresh cadence and latency (with the latency assessment if near-real-time), data quality expectations, and access roles per the [access model](../governance/access-model.md).
3. **Data contract.** Period, grain, dimensions, measures, metrics, per [Analytical dataset language](analytical-dataset-language.md). The grain is a plain-English sentence ("one row per order line per calendar day"); if the sentence is ambiguous, the spec is not ready. SCD strategy and semi-additive labels are declared here.
4. **Acceptance criteria.** Measurable, executable checks. For datasets, the pivot test is mandatory; add reconciliation targets (row counts, control totals against source), EXPECT constraint thresholds, and metric definition sign-off.
5. **Validation plan.** How acceptance runs in CI: the smoke job and acceptance checks the `nonprod` deploy executes, and what a human verifies before the `prod` gate. "Done" means these pass on the promoted SHA.

## Where specs live

- In the domain repo, under `specs/`, one Markdown file per data product or platform change, named after the thing it specifies (`sales_daily.md` for `gold.sales_daily`).
- The spec merges before the implementation PR. Reviewers approve the contract first, then the code against it.
- The spec is versioned with the code it governs. A contract change (grain, metric definition, SCD strategy) is a spec change first, in its own PR, with the version boundary documented per the metric rules.

## Workflow

1. Write the spec; name the decision, the grain, and the acceptance checks.
2. Review: data owner approves the contract; platform reviews naming, layers, and access against this repo's standards. Reuse check: does a certified definition already cover this? Extend, don't fork.
3. Build against the spec in a feature branch. Deviations found while building go back into the spec, not into silent code divergence.
4. Validate: PR runs `bundle validate` and tests; merge deploys `nonprod` and runs the spec's acceptance checks.
5. Promote: the `prod` gate reviewer reads the spec's validation plan, not the diff, to decide.
6. Certify only decision-grade outputs whose spec names the decision, decision-maker, and follow-on action ([BI practices](bi-practices-guidance.md)).

## Rules

- No implementation PR merges without a merged spec. "Exploration" in dev needs no spec; anything that MAY be promoted does.
- Acceptance criteria are executable. A criterion that cannot run in CI or be checked by a named human step is a wish, not a criterion.
- The spec is the tiebreaker. When code and spec disagree, one of them is fixed in the same PR; they never coexist disagreeing.
- Agents building against a spec cite the spec section for each design choice and surface conflicts between the spec and this repo's standards instead of resolving them silently.
- A metric with unresolved definitional questions is not specified; it is a question. Resolve entity resolution, currency, and governing timestamp before the spec merges ([BI practices](bi-practices-guidance.md)).

## Sharp edges

- Spec drift is worse than no spec: a stale spec is an authoritative-looking lie. The update rule is absolute and enforced in review.
- Rubber-stamp specs: if every spec is approved unchanged in minutes, the review is theater. The contract review is where grain mixing and metric forks die; treat it as the highest-leverage hour in the delivery cycle.
- Over-specification: a spec that prescribes implementation detail (cluster sizes, code structure) rots immediately and crowds out the contract. Specify the what and the checks; leave the how to the build and the platform docs.
- Retrofitting: writing the spec after the build to pass review produces acceptance criteria shaped to the code. The merge-order rule (spec first) exists to make this visible in history.

## Checklist

- [ ] Spec merged before implementation PR, in `specs/`, named after the object
- [ ] Decision, decision-maker, and follow-on action named, or explicitly informational
- [ ] Grain stated as an unambiguous sentence; contract complete per the dataset language
- [ ] Acceptance criteria executable; pivot test included for datasets
- [ ] CI runs the validation plan on `nonprod`; the `prod` gate reads it
- [ ] Contract changes shipped as spec changes with documented version boundaries

## References

- [Analytical dataset language](analytical-dataset-language.md): contract elements
- [BI practices guidance](bi-practices-guidance.md): decision-first design, certification
- [Medallion data practices](medallion-data-practices.md): layer rules the build must honor
- [CI/CD and deployment](../platform/cicd-and-deployment.md): where acceptance runs
