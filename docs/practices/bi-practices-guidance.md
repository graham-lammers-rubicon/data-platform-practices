# BI Practices: Decision Systems, Not Reporting

The standards that turn a reporting stack into a decision system. Priority order matters: the semantic meaning of business data comes first, then the decision workflow, then the infrastructure that serves both. Most BI guidance gets this backward and produces well-engineered platforms that inform without ever changing an outcome.

## What this covers

- Design, implementation, and operation of BI and analytics on this platform
- Certification: informational vs. decision-grade outputs
- The semantic layer: measures, metrics, definitions, versioning
- Decision-first design, the trust chain, latency, history, self-service, executive controls

"Must" is a requirement, "should" is a strong recommendation, "may" is an option.

---

## 1. The Standard: Insight, Action, Defense

Dashboards are informational by default. Informational is the floor, not the goal. A BI platform earns its cost when it does three things reporting alone cannot:

1. **Drive insight.** Explain why a number changed, not just that it changed.
2. **Drive action.** Point to the segment, account, or process that needs intervention, and to the owner who intervenes.
3. **Defend decisions.** Carry the evidence trail: the data, the definition, and the logic behind a call, so the decision holds up when it is questioned later.

Reporting that stops at "here is the number" pushes the hard work (interpretation, reconciliation, justification) onto the consumer. That work then happens in spreadsheets, hallways, and meeting prep, outside governance and outside the platform.

### 1.1 Classify every output

Every published output must be classified as **informational** or **decision-grade**:

- **Informational:** context, monitoring, general awareness. Permitted, but must be labeled as such and must not be certified.
- **Decision-grade:** names the decision it serves, the decision-maker, the action that follows, and the governed metric definitions it depends on. Only decision-grade outputs may be certified.

If no one can state the decision an output serves, it is informational. That is not a failure. Citing an informational output as the basis for a decision is.

### 1.2 Success measures

The primary measure of the platform is whether it supports better, faster, more accountable decisions. The following are operational indicators only and must not be used as the measure of success:

- Number of dashboards created
- Number of data sources connected
- Report refresh speed
- Number of users with self-service access

### 1.3 Failure indicators

These behaviors mean the platform is informing but not deciding, regardless of infrastructure quality. Treat them as defects:

- Decision-makers export data to spreadsheets rather than acting on platform outputs
- Teams reconcile conflicting figures manually before meetings
- Departments disagree on the definition of shared KPIs
- Business teams depend on analysts to explain routine metric changes

### 1.4 Measure the platform itself

"Better decisions" is not measurable by assertion. The platform team must instrument and review its own decision-support performance:

- [ ] Certified vs. informational output usage, per domain
- [ ] Exports from certified outputs (defect signal, target zero)
- [ ] Time from metric request to certified definition
- [ ] Per output: the documented decision is still live; retire it if not

These indicators are how the primary measure in 1.2 gets observed. A platform that mandates outcome-based success criteria for consumers but does not instrument its own is exempting itself from this document.

---

## 2. Semantic Meaning Comes First

The most valuable thing the platform produces is not a dashboard. It is agreed meaning: what a customer is, what counts as revenue, when a renewal is a renewal, which timestamp governs. Infrastructure moves data. The semantic layer is what makes the data mean something, and meaning must be defined once, owned, and versioned.

Without a governed semantic layer, each report author applies independent interpretations: one filters cancelled transactions, another includes them; one uses invoice dates, another payment dates. Every report is individually defensible and the set is mutually inconsistent. That inconsistency is what kills trust, and trust is what makes a number decision-grade.

### 2.1 Measures vs. metrics

This distinction is where semantic layers break down. Do not conflate them.

- **Measure:** a raw stored fact in a fact table. Stored in Silver at source grain.
- **Metric:** a governed business calculation defined once in Gold, with a name, plain-English definition, aggregation rule, filters, time intelligence, grain, owner, and version.

You store measures. You define metrics. You report metrics to the business. One definition per metric name. Metrics reference measures, never other metrics. See the medallion practices doc for the storage rules.

### 2.2 Semantic layer requirements

The semantic layer must provide:

- [ ] Business entity definitions (customer, active user, revenue, backlog, renewal, churn)
- [ ] Shared KPI calculations, referenced not copied
- [ ] One rate source for currency; period boundaries from the conformed period dimension
- [ ] A named owner per metric
- [ ] A rule for late and corrected data: restate, append, or version
- [ ] Version history for every definition change
- [ ] Metric-to-source traceability

Two checks: lineage shows consumers reading governed definitions, never re-implementing them; two certified metrics converting the same amount on the same date return the same value.

The semantic layer must be independent of the visualization tool. Definitions that live inside a BI tool are locked to that tool and diverge the moment a second tool appears.

### 2.3 Metric definition completeness

A metric is not defined until its edge cases are. Example: "outstanding receivables" must specify:

- [ ] Customer record resolution across source naming conventions
- [ ] Treatment of disputed invoices and partial payments
- [ ] Handling of customer ownership changes over time
- [ ] Currency conversion method and rate source
- [ ] The governing timestamp (invoice, payment, or warehouse arrival)
- [ ] The threshold where an overdue invoice becomes operationally significant

A metric with unresolved definitional questions must not be certified for decision use. An ambiguous metric on a fast dashboard is a fast way to make a wrong call with confidence.

### 2.4 Definition changes are versioned events

When a metric definition changes, the version boundary must be documented, consumers notified, and historical reporting must state which version produced which figures. A silent definition change invalidates every trend line built on the old one.

### 2.5 Reuse before build

Before authoring a new metric, entity definition, or dataset, teams must check the semantic layer for an existing certified definition. If one exists, use it or extend it; do not create a parallel definition for a new use case. A new definition is justified only when the business meaning is genuinely different, and it must then carry a different name. Two names for the same meaning is a defect. One name for two meanings is worse.

Extending is the normal path: domain-specific attributes and filters may build on a shared definition, but the shared keys, base measure references, and aggregation rules must not diverge. If a proposed extension changes any of those, it is a new version of the shared definition and goes through the owner, not a fork.

---

## 3. Design Backward from the Decision

Platform and report design must begin with the decision and the decision-maker, not with the dashboard. Before designing metrics, semantic definitions, transformations, ingestion, or visualization, document:

- [ ] The decision and its owner
- [ ] The decision frequency
- [ ] The inputs that change the outcome
- [ ] The maximum data age before the decision degrades
- [ ] The smallest error that changes the decision, and the costlier direction
- [ ] The exceptions that block the decision, and who resolves them
- [ ] The actions that follow
- [ ] The outcome measure and when it is checked

Answers that fail review: "all available data," "real-time" (route through Section 5), and "100% accurate." Each one defers the tradeoff to the pipeline builder.

Designing forward from infrastructure or visualization produces disconnected components that inform. Designing backward from the decision ensures every layer traces to a decision requirement. The test for the platform as a whole: it must reduce uncertainty at the moment of decision.

---

## 4. The Trust Chain

A dashboard is the final interface of a larger system and can only be as trusted as the layers beneath it. A decision defended with a number is only as defensible as that chain. For every certified metric, the platform must document:

- [ ] Ingestion path from source system to platform
- [ ] Matching logic that resolves records across sources
- [ ] Business rules applied, and the layer that applies them
- [ ] Mechanism preserving historical attribute changes
- [ ] Calculation producing the metric value
- [ ] Handling of exceptions and corrections
- [ ] Access controls on the data
- [ ] Latency from source event to availability

If any link is undocumented, the metric is informational, not decision-grade.

---

## 5. Latency Follows Decision Cadence

The purpose of low-latency analytics is to shrink the gap between an event and the organization's response, not to maximize refresh frequency. Latency requirements must be derived from decision cadence.

Before implementing real-time or near-real-time delivery for a metric, document:

- [ ] **Decision speed:** the time from event to required decision
- [ ] **Cost of partial data:** the impact of acting before corrections arrive
- [ ] **Response capability:** the fastest cadence the consuming team can act on

Calibration: fraud alerts need milliseconds, collections runs daily, board metrics move once per period. A per-minute refresh is not justified by weekly action.

Real-time delivery must not be enabled for data with unresolved inconsistencies, duplication, late-arriving events, or incomplete mappings. High refresh frequency on unreliable data is false precision: it makes a shaky number look authoritative, which is worse than a slow number that is right.

---

## 6. History Is Evidence

You cannot defend a prior-period decision using current attribute values. Evaluating last year's territory performance with this year's territory assignments misstates history and makes the resulting numbers indefensible.

Analytical systems must preserve historical states of dimensional attributes rather than overwriting them. Overwrite-in-place is acceptable for operational applications, not for analytics.

The criterion: SCD Type 2 (or an equivalent mechanism preserving each attribute state and its validity interval) must be applied to any attribute used to evaluate past performance, determine compliance, or calculate compensation. Examples: sales territory assignment, customer segment, product category and cost basis, account ownership, contract terms. The list of qualifying attributes grows over time; the criterion does not. If an attribute appears in a prior-period comparison anywhere, it qualifies.

**Acceptance criterion:** the platform must be able to answer "what did the business look like at time T" for any governed dimension.

---

## 7. Entity Resolution Is a Business Decision

Where entity resolution is required (a Customer 360 across sales, billing, support, product usage, marketing, and third-party sources), exact matching is insufficient when the same entity appears under different legal entities, accounts, subsidiaries, or identifiers. Fuzzy matching and entity resolution algorithms may be used, but the meaning of a match is a business decision, not a model output. The business owner, not the algorithm, must decide:

- [ ] The confidence threshold for automatic match processing
- [ ] The conditions that route a record to human review
- [ ] The procedure for splitting a false merge
- [ ] The system-of-record precedence when sources conflict

---

## 8. Self-Service Is Controlled Freedom

Self-service reduces dependence on centralized BI teams for routine exploration. Ungoverned self-service replaces one reporting bottleneck with many inconsistent reports, and the inconsistency lands in the same meetings the platform was supposed to fix. The standard is controlled freedom: users explore certified data without rebuilding business logic.

Self-service environments must separate three layers:

1. **Data engineering layer:** ingestion, transformation, quality checks, orchestration, reliability
2. **Governed analytics layer:** shared entity definitions, metric certification, security rules, analytical models
3. **Exploration layer:** filtering, comparison, and visualization by business users

Granting business users direct access to raw tables must not be classified as self-service. It is distributed data engineering without standards.

---

## 9. Executive Metrics Carry Elevated Controls

Metrics used in investor communications, earnings, board meetings, and strategic planning are decisions being defended in public. They are subject to stricter controls than operational dashboards:

- [ ] Documented and explained adjustments
- [ ] A published refresh schedule
- [ ] The same value for every authorized user

Executive reporting must not depend on manual spreadsheets, ad-hoc filters, one-off extracts, or other unrepeatable methods. For every executive metric, the platform must produce on demand:

- [ ] The origin of the number, end to end
- [ ] The source records used in the calculation
- [ ] The business rules applied
- [ ] The timestamp of the last refresh
- [ ] Logic changes since the previous reporting period
- [ ] The approver of the metric definition
- [ ] The procedure that reproduces the result

---

## 10. Automation Moves Attention to Judgment

Automation must redirect analyst attention from data preparation to decision analysis. Time saved is a secondary measure; the primary measure is the shift in where attention is spent.

### 10.1 Automate the mechanical work

The platform must eliminate recurring manual work including: file downloads, column reformatting, copying values between systems, reconciling repeated totals, updating presentation slides, repairing spreadsheet formulas, and explaining discrepancies between dashboards.

### 10.2 Preserve the judgment

Automation must not remove human judgment from decisions requiring business context. The intended analyst focus after automation: why a metric changed, which segments drive the change, whether the change is temporary or structural, which accounts require intervention, and which actions have the greatest impact. That list is the insight-and-action work from Section 1. If automation does not free capacity for it, the automation optimized the wrong thing.

### 10.3 Anomaly alerts must carry context

Anomaly detection identifies deviations; it does not explain them. An anomaly may reflect fraud, a technical malfunction, seasonality, contract renewals, pricing changes, or legitimate growth. Alerts without context create work; alerts with context start investigations. Alerts must include:

- [ ] Expected interval and observed value
- [ ] Deviation magnitude and comparison period
- [ ] Localizing dimensions (segment, region, channel)
- [ ] Data quality status of contributing pipelines
- [ ] Past occurrences and their resolution
- [ ] The records driving the deviation

---

## 11. Governance Lives in the Platform

Governance must be enforced by the platform architecture, not by policy documents and committees. The compliant path must be the default path; compliance must not depend on individuals remembering policy.

The control catalog is not restated here. Where each control lives, and how to check it:

- **Access, environments, secrets, deployments:** the [access model](../governance/access-model.md) and the platform reference docs (`docs/platform/`). This doc adds one rule: consumers read certified Gold only (Section 8). Check: no consumer grant below Gold.
- **Certification and labeling:** Section 1.1 and the acceptance checklist. The label lives in the catalog as governed metadata. Check: queryable for every published output.
- **Lineage and audit:** the trust chain (Section 4) and executive audit items (Section 9). Check: every item producible from platform metadata alone, no humans required.

As more operational systems connect to centralized analytics and data movement gets easier, governance of definitions, access, and usage becomes more critical, not less.

---

## Sharp edges

- **Informational outputs cited in decisions.** The most common failure mode. The dashboard was never certified, nobody noticed, and now a board number traces to it.
- **Definitions living in the BI tool.** They diverge silently the moment a second tool or a second author appears. One definition, in the semantic layer, tool-independent.
- **Silent definition changes.** A changed metric definition without a version boundary corrupts every trend built on the old one, and nobody can say when the break happened.
- **False precision.** Real-time refresh on unreconciled data makes wrong numbers look authoritative. Latency must follow decision cadence, not technical capability.
- **Overwritten history.** Current attribute values applied to prior periods make executive and regulatory numbers indefensible. There is no retroactive fix; the history is gone.
- **Raw-table self-service.** It demos well and decays into distributed data engineering with no standards, surfacing as KPI disagreements months later.
- **Context-free anomaly alerts.** They train users to ignore alerts, which defeats the reason for having them.

## Acceptance checklist

A published output is decision-grade only if all of the following hold:

- [ ] Decision, decision-maker, and follow-on action documented
- [ ] One governed definition per metric, with owner and version
- [ ] Each metric reuses or extends an existing definition, or documents why its meaning is new (Section 2.5)
- [ ] Definitional edge cases resolved (Section 2.3)
- [ ] Latency assessment documented and matched to decision cadence
- [ ] History preserved for governed dimensions; the time-T state is answerable
- [ ] All eight trust chain items documented (Section 4)
- [ ] The same value for every authorized user, reproducibly
- [ ] No manual spreadsheet step between platform and decision

Anything that fails this checklist may still be published, labeled informational. It must not be certified.

## Sources

- Kimball Group: [Dimensional modeling techniques](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) (conformed dimensions, slowly changing dimensions)
- Internal: [Analytical dataset language](analytical-dataset-language.md), [Medallion data practices](medallion-data-practices.md), [Access model](../governance/access-model.md), [Spec-driven development](spec-driven-development.md)

