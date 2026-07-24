# BI Platform Guidance and Standards

**Scope:** Design, implementation, and operation of enterprise business intelligence and analytics platforms.
**Normative language:** "Must" indicates a requirement. "Should" indicates a strong recommendation. "May" indicates an option.

---

## 1. Success Criteria

### 1.1 Primary measure

The primary measure of a BI platform is whether it supports better, faster, more accountable decisions. Platform teams must define success criteria in terms of decision outcomes, not delivery volume.

### 1.2 Secondary measures

The following are operational indicators only and must not be used as the sole measure of platform success:

- Number of dashboards created
- Number of data sources connected
- Report refresh speed
- Number of users with self-service access

### 1.3 Indicators of failure

The following behaviors indicate the platform is not functioning as a decision system, regardless of infrastructure quality, and must be treated as defects:

- Decision-makers export data to spreadsheets rather than using platform outputs
- Teams reconcile conflicting figures manually before meetings
- Departments disagree on the definition of shared KPIs
- Business teams depend on analysts to explain routine metric changes

---

## 2. Platform Architecture Requirements

A dashboard is the final interface of a larger system and can only be as trusted as the layers beneath it. For every published metric, the platform must have explicit, documented answers for:

1. How the data is ingested
2. How records are matched across sources
3. How business rules are applied
4. How historical changes are stored
5. How the metric is calculated
6. How exceptions are processed
7. How access is secured
8. How quickly data becomes available

### 2.1 Metric definition completeness

Every published metric must resolve its underlying definitional questions before release. Example, for "outstanding receivables," the definition must specify:

- Customer record resolution when source systems use different naming conventions
- Treatment of disputed invoices
- Treatment of partial payments
- Handling of customer ownership changes over time
- Currency conversion method and rate source
- The governing timestamp (invoice date, payment date, or warehouse arrival)
- The threshold at which an overdue invoice becomes operationally significant

A metric with unresolved definitional questions must not be certified for decision use.

---

## 3. Data Latency Standards

### 3.1 Principle

The purpose of low-latency analytics is to reduce the gap between an event and the organization's response, not to maximize refresh frequency. Latency requirements must be derived from decision cadence.

### 3.2 Required assessment

Before implementing real-time or near-real-time delivery for a metric, teams must document:

1. **Decision speed:** How quickly must the resulting decision be made? (e.g., fraud alerts in milliseconds; collections prioritization daily; board metrics once per reporting period)
2. **Cost of partial data:** What is the impact of acting on incomplete data — for example, revenue shown before adjustments, cancellations, or reconciliations arrive?
3. **Organizational response capability:** Can the consuming team act at the refresh frequency? A metric refreshed every minute is not justified if action occurs weekly.

### 3.3 Data quality precondition

Real-time delivery must not be enabled for data with unresolved inconsistencies, duplication, late-arriving events, or incomplete mappings. High refresh frequency on unreliable data creates false precision.

---

## 4. Semantic Layer Standards

### 4.1 Requirement

The platform must include a governed semantic layer that translates raw records into standardized business concepts (e.g., customer, active user, revenue, backlog, renewal, churn, risk, profit margin). Without one, each report author applies independent interpretations — one filtering cancelled transactions, another including them; one using invoice dates, another payment dates — producing reports that are individually defensible but mutually inconsistent.

### 4.2 Required capabilities

The semantic layer must provide:

- Definitions of business entities
- Shared calculation logic for common KPIs
- Consistent treatment of time and currency
- Named ownership for each metric definition
- Defined handling of late-arriving and corrected data
- Tracking of definitional changes over time
- Traceability from executive metrics to source records

### 4.3 Tool independence

The semantic layer must be independent of the visualization tool, so that visualization tools can be replaced without changing metric logic.

---

## 5. Data Modeling and Entity Resolution

### 5.1 Principle

Data modeling choices (schema design, fact/dimension definitions, dimension change handling, aggregation) encode business decisions and must be reviewed as such, not treated as purely technical.

### 5.2 Entity resolution governance

Where entity resolution is required (e.g., Customer 360 across sales, billing, support, product usage, marketing, and third-party sources), exact matching is insufficient when the same entity appears under different legal entities, accounts, subsidiaries, or identifiers. Fuzzy matching and entity resolution algorithms may be used, subject to the following governance requirements. The business owner, not the algorithm, must decide:

- The confidence threshold at which a match is processed automatically
- The conditions under which a record is routed for human review
- The procedure for splitting merged records when a match proves false
- The system-of-record precedence when sources conflict

---

## 6. Historical Data Management

### 6.1 Requirement

Analytical systems must preserve historical states of dimensional attributes rather than overwriting them. Overwrite-in-place (e.g., updating a customer's region, sales rep assignment, or product category) is acceptable for operational applications but not for analytics: evaluating prior-period performance using current attribute values misstates history.

### 6.2 Standard mechanism

Slowly Changing Dimension Type 2 (or an equivalent mechanism preserving each attribute state and its validity interval) must be applied to attributes used in:

- Sales territory performance evaluation
- Customer segmentation
- Product profit margin analysis
- Ownership and assignment reporting
- Regulatory reporting
- Contract analytics
- Forecast evaluation
- Executive compensation calculations

### 6.3 Acceptance criterion

The platform must be able to answer "what did the business look like at time T" for any governed dimension.

---

## 7. Self-Service Analytics Standards

### 7.1 Principle

Self-service reduces dependence on centralized BI teams for routine exploration. Ungoverned self-service replaces one reporting bottleneck with many inconsistent reports. The standard is controlled freedom: users explore certified data without rebuilding business logic.

### 7.2 Required layer separation

Self-service environments must separate responsibilities into three layers:

1. **Data engineering layer:** Ingestion, transformation, quality checks, orchestration, reliability
2. **Governed analytics layer:** Shared entity definitions, metric certification, security rules, analytical models
3. **Exploration layer:** Filtering, comparison, and visualization by business users

### 7.3 Prohibited pattern

Granting business users direct access to raw tables must not be classified as self-service. It constitutes distributed data engineering without standards.

---

## 8. Executive Reporting Controls

### 8.1 Elevated requirements

Metrics used in investor communications, earnings, board meetings, and strategic planning are subject to stricter controls than operational dashboards:

- Numbers must be reproducible
- Metric logic must be traceable
- Adjustments must be documented and explained
- Refresh schedules must be predictable
- Access must be controlled
- The same metric must produce the same value for every authorized user

### 8.2 Prohibited practices

Executive reporting must not depend on manual spreadsheets, ad-hoc filters, one-off extracts, or other unrepeatable methods.

### 8.3 Auditability requirements

For every executive metric, the platform must be able to answer:

- Where does this number come from?
- Which source records were used in its calculation?
- Which business rules were applied?
- When was the data last refreshed?
- Has the logic changed since the previous reporting period?
- Who approved the metric definition?
- How can the result be reproduced?

---

## 9. Automation Standards

### 9.1 Objective

Automation must redirect analyst attention from data preparation to decision analysis. Time saved is a secondary measure; the primary measure is the shift in where attention is spent.

### 9.2 Tasks to automate

The platform must eliminate recurring manual work including: file downloads, column reformatting, copying values between systems, reconciling repeated totals, updating presentation slides, repairing spreadsheet formulas, and explaining discrepancies between dashboards.

### 9.3 Preserved human judgment

Automation must not remove human judgment from decisions requiring business context. The intended analyst focus after automation includes: why a metric changed, which segments drive the change, whether the change is temporary or structural, which accounts require intervention, and which actions have the greatest impact.

---

## 10. Anomaly Detection Standards

### 10.1 Principle

Anomaly detection identifies deviations; it does not explain them. An anomaly may reflect fraud, technical malfunction, seasonality, contract renewals, pricing changes, or legitimate growth. Alerts without context create work; alerts with context start investigations.

### 10.2 Required alert context

Anomaly alerts must include:

- Expected interval
- Observed value
- Deviation magnitude
- Historical comparison period
- Related business dimensions
- Possible data quality causes
- Similar past events
- Contributing records

---

## 11. Embedded Governance Requirements

### 11.1 Principle

Governance must be enforced by the platform architecture, not solely by policy documents and committees. The compliant path must be the default path; compliance must not depend on individuals remembering policy.

### 11.2 Required platform enforcement

The platform must enforce:

- Role-based access control
- Row-level and column-level security
- Data lineage
- Metric certification
- Data quality validation
- Retention rules
- Audit logging
- Environment separation
- Controlled deployments
- Ownership metadata

### 11.3 Scaling consideration

As more operational systems connect to centralized analytics and data movement becomes easier, governance of definitions, access, and usage becomes more critical, not less.

---

## 12. Design Process: Work Backward from the Decision

### 12.1 Required starting point

Platform and report design must begin with the decision and the decision-maker, not with the dashboard. Before designing metrics, semantic definitions, transformations, ingestion, or visualization, teams must document:

1. What decision will be made?
2. How frequently is it made?
3. What information influences it?
4. How current must the information be?
5. What accuracy is required?
6. How should exceptions be explained?
7. What actions follow from the insight?
8. How will the outcome be measured?

### 12.2 Rationale

Designing forward from infrastructure or visualization produces disconnected components. Designing backward from the decision ensures every platform layer traces to a decision requirement. The standard for the platform as a whole: it must reduce uncertainty at the moment of decision.
