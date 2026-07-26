# Analytical Dataset Language: Period, Grain, Dimensions, Measures, and Metrics
## A Cross-Domain Dataset Design Reference

---

## Purpose

A shared analytical dataset language gives every domain (Sales, Finance, HR, Marketing, Operations) a common structural contract so their datasets can be queried, compared, and forecasted with identical patterns, without translation.

This document defines five elements that compose that contract:

- **Period**: the time axis every fact table attaches to
- **Grain**: the row-level contract that must be declared before anything else
- **Dimensions**: the context axes used for filtering and grouping
- **Measures**: the raw numeric facts stored at the declared grain
- **Metrics**: the governed business calculations built on top of measures

> **Falsified if:** a dataset cannot be pivoted (rows to columns across a period, measures at the intersection), the structure is broken. Fix the structure before building anything on top of it.

---

## The Five Elements

### 1. Period

> Period is the time axis. It is not just a date column. It is a first-class dimension with a declared grain and a hierarchy. Every fact table must attach to a period. No exceptions.

**Period grain options (from atomic to aggregate):**

| Grain | Example value | Use case |
|---|---|---|
| Instant | `2025-03-14 09:31:47` | Event logs, transactions |
| Day | `2025-03-14` | Default atomic grain for most domains |
| Week | `2025-W11` | Operational reporting |
| Month | `2025-03` | Financial snapshots, HR headcount |
| Quarter | `2025-Q1` | Exec reporting, budgeting |
| Fiscal period | `FY2025-P6` | When fiscal calendar differs from calendar |
| Year | `2025` | Strategic aggregations |

**Period hierarchy (roll-up path):**

```
Day → Fiscal Week → Fiscal Month → Fiscal Quarter → Fiscal Year
Day → Calendar Week → Calendar Month → Calendar Quarter → Calendar Year
```

**Rules:**

- Period grain must be declared before identifying any dimension or measure. The grain is the contract.
- Store the atomic grain. Aggregation is cheap at query time; re-atomizing is expensive or impossible.
- Every fact table gets at least one `period_key`. Some get multiple (e.g., order date + ship date + return date).
- Period is a conformed dimension: its keys, attributes, and hierarchy must be identical across every domain that uses it.

**Period table (representative columns):**

```sql
period (
  period_key          INT PRIMARY KEY,   -- surrogate, e.g. 20250314
  calendar_date       DATE,
  day_of_week         VARCHAR,
  week_of_year        INT,
  iso_week            VARCHAR,           -- e.g. 2025-W11
  calendar_month      INT,
  calendar_month_name VARCHAR,
  calendar_quarter    INT,
  calendar_year       INT,
  fiscal_period       INT,              -- domain-defined
  fiscal_quarter      INT,
  fiscal_year         INT,
  is_weekend          BOOLEAN,
  is_company_holiday  BOOLEAN
)
```

---

### 2. Grain

Grain is the most consequential decision in dimensional design. It defines what one row in a fact table represents. Everything else flows from the grain declaration: which dimensions attach, which measures are valid, how aggregation behaves.

> **Grain is a contract, not a preference.** Once declared, it cannot be violated without creating silent errors. Mixed grain in a single fact table is the most common source of double-counting in production BI systems.

**The grain declaration must be written in plain English before any physical design begins:**

```
"One row per order line, per calendar day."
"One row per product per location, snapshot at end of each calendar month."
"One row per GL journal posting, at the moment of posting."
"One row per employee per department assignment, snapshot at last day of each month."
```

> If you cannot write this sentence unambiguously, the design is not ready.

**The three fundamental grain types:**

| Type | Pattern | Row is created when | Typical measures |
|---|---|---|---|
| **Transaction** | One row per discrete event | The event occurs | Additive quantities and amounts |
| **Periodic Snapshot** | One row per entity per period, regardless of activity | The period closes | Semi-additive balances and counts |
| **Accumulating Snapshot** | One row per entity lifecycle, updated as milestones pass | Entity created; updated at each milestone | Lag times, completion flags, pipeline stage |

**Transaction grain:** most dimensional and most expressive. Maximum slicing and dicing. Rows exist only when something happens, so the table is often sparse.

```
Example: sales_orders
Grain: one row per order line per day
A row exists only when a line item is ordered.
```

**Periodic snapshot grain:** rows are inserted even when no activity occurs. This keeps trend analysis clean: every period has a value, even if it is zero or unchanged.

```
Example: inventory_snapshot
Grain: one row per product per location per calendar day
A row exists for every product-location combination every day,
whether or not any inventory movement occurred.
```

**Accumulating snapshot grain:** one row per entity (order, claim, application, patient episode), updated as milestones are reached. Multiple date foreign keys, one per milestone. Measures include lag times between milestones.

```
Example: order_fulfillment_pipeline
Grain: one row per order
Columns: order_date_key, pick_date_key, pack_date_key, ship_date_key, deliver_date_key
Measures: days_to_pick, days_to_ship, days_to_deliver
Row is inserted at order creation and updated as each milestone completes.
```

**Rules:**

- Declare grain before touching schema. Write the sentence. Get agreement. Lock it.
- Never mix grains in a single fact table. If two proposed measures imply different grains, they belong in different tables.
- The grain determines which fact table type to use. The type determines how measures aggregate across time.
- > Atomic grain is almost always the right starting point. You can always roll up; you cannot roll down.
- Document the grain as a table comment in the physical schema. Make it impossible to miss.

**Grain and the pivot test:** A fact table with a clean grain declaration will pivot cleanly. If a pivot produces unexpected row multiplication or aggregation errors, the grain has been violated somewhere in the pipeline.

---

### 3. Dimensions

Dimensions provide the "who, what, where, why, and how" context for every measurement. They are the filtering and grouping axes of every analytical query.

**Properties of a well-formed dimension:**

| Property | Requirement |
|---|---|
| Conformed | Same key, same attribute names, same domain values across all fact tables that reference it |
| Atomic | Stored at lowest granularity; hierarchies are attributes on the same row |
| Stable key | Surrogate key that survives source system changes |
| Self-describing | Attributes are human-readable without joining to another table |
| SCD-aware | Change strategy declared (Type 1 overwrite, Type 2 versioned row, Type 3 prior value column) |

**Core conformed dimensions (cross-domain):**

```
period       → when
entity       → who (customer, employee, patient, student — shared key, domain-specific attributes)
product      → what (SKU, service, course, procedure — shared key, domain-specific attributes)
location     → where (site, region, territory, cost center, facility)
channel      → how the event occurred (web, in-store, phone, EDI, mobile)
org          → internal owner (department, cost center, business unit, team)
```

**Domain-specific dimensions (not conformed, but patterned consistently):**

```
campaign     → Marketing
supplier     → Procurement
provider     → Healthcare
course       → Education
incident     → Operations / IT
```

**Dimension hierarchy example, Location:**

```
Building → Site → City → Region → Country → Global
```

Stored flat (denormalized) in a single row:

```sql
location (
  location_key    INT PRIMARY KEY,
  location_id     VARCHAR,          -- natural key from source
  building_name   VARCHAR,
  site_name       VARCHAR,
  city            VARCHAR,
  state_province  VARCHAR,
  country         VARCHAR,
  region          VARCHAR,
  territory       VARCHAR
)
```

---

### 4. Measures

Measures are the numeric facts at the intersection of dimensions and period. They are what you analyze.

**Measure classification:**

| Class | Definition | Examples | Aggregation |
|---|---|---|---|
| **Additive** | Can be summed across all dimensions | Revenue, quantity, headcount events | SUM |
| **Semi-additive** | Can be summed across some dimensions but not period | Account balance, inventory on hand, headcount snapshot | SUM across entity, AVG across time |
| **Non-additive** | Cannot be summed at all | Ratios, percentages, rates | Must be recomputed from components |
| **Derived** | Computed from other measures | Margin = Revenue - COGS, Fill Rate = Shipped / Ordered | Compute at query time. Never store a ratio in a Silver fact table; Gold may expose one at its declared grain with components alongside |

**Rules:**

- Store additive components, not derived ratios. Compute ratios at the presentation layer.
- Every measure in a fact table must be at the same grain. Mixed grain in a single table is a design defect.
- Name measures so the aggregation is unambiguous: `revenue_usd`, `order_count`, `headcount_eop` (end of period).
- Unit of measure is part of the column name or metadata. Never store mixed currencies in one column without a currency dimension.

---

### 5. Metrics

A metric is not a measure. This distinction is where most semantic layer implementations break down.

> **Measure:** a raw stored fact in a fact table. Additive, semi-additive, or non-additive. Lives in the physical data layer.
>
> **Metric:** a defined business calculation built on top of one or more measures. Has an explicit aggregation rule, optional filters, optional time intelligence, and a governed definition. Lives in the semantic layer.
>
> You store measures. You define metrics. You report metrics to the business.

**Why the distinction matters:**

A measure can be summed. A metric may require a specific aggregation path that is not simply SUM. If you expose raw measures to BI consumers without a metric layer, each analyst will implement their own version of `churn_rate` or `LTV` and they will not match. The metric layer is where that definition lives once, is tested once, and is trusted everywhere.

**Metric anatomy:**

| Component | Description | Example |
|---|---|---|
| **Name** | Business-facing label, not a column name | `Monthly Recurring Revenue` |
| **Definition** | Plain English business rule | Sum of active subscription ARR / 12, excluding trials |
| **Measure inputs** | Which raw measures it consumes | `contract_arr`, `subscription_status` |
| **Aggregation** | How it rolls up | SUM of MRR across active subscriptions |
| **Filters** | Conditions applied before aggregation | `subscription_status = 'ACTIVE'` |
| **Time intelligence** | Period behavior | Point-in-time snapshot (semi-additive); MTD, QTD, YTD variants |
| **Grain** | Finest level at which the metric is meaningful | Per customer per month |
| **Owner** | Team or role responsible for the definition | Finance / RevOps |
| **Version** | When the definition changed and why | v2 effective FY2025-Q1: excludes pilot accounts |

**Metric classification:**

| Class | Description | Example |
|---|---|---|
| **Volume** | Count or sum of activity | Orders placed, revenue booked |
| **Rate** | Ratio of two measures | Conversion rate, fill rate, churn rate |
| **Intensity** | Measure per unit of another dimension | Revenue per employee, cost per click |
| **Stock** | Point-in-time balance or count | Headcount, ARR, inventory on hand |
| **Flow** | Change in a stock over a period | New ARR, churned ARR, net headcount change |
| **Composite** | Multi-step formula combining multiple sources | LTV = ARPU × (1 / churn_rate); NRR = (Expansion + Renewal - Churn) / Prior ARR |

**Cross-domain metric example, Customer Acquisition Cost (CAC):**

CAC spans two domains (Marketing and Sales) and cannot be defined from either domain alone.

```
CAC = (Marketing Spend + Sales Fully-Loaded Cost) / New Customers Acquired

Inputs:
  marketing_spend.spend_usd          (Marketing domain)
  hr_headcount.salary_cost_usd       (HR domain, filtered to Sales org)
  sales_orders.entity_key            (Sales domain, filtered to first order per entity)

Filter: new customers only (first order date = period date, no prior orders)
Time intelligence: rolling 3-month average preferred; single-month is noisy
Grain: monthly, by channel
Owner: RevOps
```

This metric requires a drill-across join. It is not computable from a single fact table. The metric definition is the place where that join is specified once, not re-derived by each analyst.

**Metric governance rules:**

- One definition per metric name. If two teams define `revenue` differently, they must use different names.
- Metrics reference measures, not other metrics, to avoid cascading definition failures. A composite formula written with metric names (LTV = ARPU × 1 / churn_rate) is shorthand: the governed definition expands each input to its component measures, so no metric depends on another metric's definition at query time.
- Every metric carries its grain. A metric defined at monthly grain cannot be queried at daily grain without an explicit restatement of the definition.
- Metric definitions are versioned. When a definition changes (fiscal calendar shift, inclusion rule change), prior periods must be recomputed or the version boundary documented.
- Non-additive metrics must declare how they aggregate across dimensions. `Conversion rate` aggregated across channels is not the average of channel conversion rates: it is total conversions / total clicks across channels.
- Derived metrics (rates, ratios, composites) must expose their numerator and denominator as separate queryable measures so consumers can verify the computation.

**Metric layer in the lakehouse stack:**

```
Bronze (raw)      →  source measures land here, no business logic
Silver (tidy)     →  conformed measures, grain declared, dimensions attached
Gold (semantic)   →  metrics defined here, aggregation rules enforced,
                     time intelligence applied, governance metadata attached
Presentation      →  BI tools, dashboards, notebooks query Gold metrics,
                     not Silver measures
```

> The Gold layer is where measures become metrics. It is not optional in a production system. Without it, every BI tool becomes its own semantic layer and definitions diverge.

On this platform, metrics are implemented as Unity Catalog metric views: the definition (dimensions, measures, filters, comments) lives in YAML on a governed UC object, aggregation happens at query time via `MEASURE()`, and Genie and AI/BI dashboards consume the same definition. See [Databricks metric views](../platform/databricks-metric-views.md).

**Pivoting metrics:** Metrics that are additive or derived from additive components pivot cleanly. Rate and composite metrics do not pivot directly. They must be computed from their component measures at each pivot grain. Document this on every non-additive metric definition.

---

## The Dataset Contract

A dataset that conforms to this language has this shape:

```
<domain>_<grain> (
  period_key        FK → period
  entity_key        FK → entity        (if applicable)
  product_key       FK → product       (if applicable)
  location_key      FK → location      (if applicable)
  channel_key       FK → channel       (if applicable)
  org_key           FK → org           (if applicable)
  [domain-specific dimension keys]

  measure_1         NUMERIC               -- additive
  measure_2         NUMERIC               -- additive
  measure_3         NUMERIC               -- semi-additive, labeled accordingly
  [additional measures at same grain]
)
```

The pivot test: **Rows = entities (or products, or locations). Columns = periods. Values = a single measure.** If the dataset cannot produce this shape with a standard GROUP BY and CASE or PIVOT, the contract is broken.

---

## Cross-Domain Dataset Examples

### Bus Matrix

The bus matrix maps business processes (rows) to conformed dimensions (columns). A checked cell means the dimension applies to that process. This is the enterprise design contract.

```
                        | period | entity | product | location | org | channel |
------------------------|--------|--------|---------|----------|-----|---------|
sales_orders            |   ✓    |   ✓    |    ✓    |    ✓     |  ✓  |    ✓    |
inventory_snapshot      |   ✓    |        |    ✓    |    ✓     |  ✓  |         |
finance_gl              |   ✓    |        |         |          |  ✓  |         |
hr_headcount            |   ✓    |   ✓    |         |    ✓     |  ✓  |         |
marketing_spend         |   ✓    |        |    ✓    |    ✓     |  ✓  |    ✓    |
support_tickets         |   ✓    |   ✓    |    ✓    |          |  ✓  |    ✓    |
```

Every row is a separate dataset with its own grain. Conformed dimensions (columns) are physically shared: same surrogate keys, same attribute definitions.

---

### Domain 1: Sales Orders (grain: order line per day)

**Grain declaration:** One row per order line, per calendar day.

**Measures:**

| Measure | Type | Notes |
|---|---|---|
| `ordered_quantity` | Additive | Units ordered |
| `shipped_quantity` | Additive | Units actually shipped |
| `revenue_usd` | Additive | Line revenue at transaction FX rate |
| `list_price_usd` | Additive | List price at time of order |
| `discount_usd` | Additive | Discount applied |
| `order_count` | Additive | Count of distinct orders (degenerate key pattern) |
| `fill_rate_pct` | Non-additive | `shipped_quantity / ordered_quantity`. Compute at query time |

**Pivotable queries:**

```sql
-- Revenue by product, pivoted by month
SELECT
  p.product_name,
  SUM(CASE WHEN pr.calendar_month = 1 THEN f.revenue_usd ELSE 0 END) AS jan,
  SUM(CASE WHEN pr.calendar_month = 2 THEN f.revenue_usd ELSE 0 END) AS feb,
  SUM(CASE WHEN pr.calendar_month = 3 THEN f.revenue_usd ELSE 0 END) AS mar
FROM sales_orders f
JOIN product p USING (product_key)
JOIN period pr USING (period_key)
WHERE pr.calendar_year = 2025
GROUP BY p.product_name;
```

---

### Domain 2: Inventory Snapshot (grain: product-location per day)

**Grain declaration:** One row per product per location per calendar day. This is a periodic snapshot: rows are inserted even when no activity occurs.

**Measures:**

| Measure | Type | Notes |
|---|---|---|
| `on_hand_units` | Semi-additive | Sum across products/locations; average across time |
| `on_order_units` | Semi-additive | Units in open POs |
| `days_of_supply` | Non-additive | `on_hand / avg_daily_demand`. Compute at query time |
| `stockout_flag` | Additive (as count) | 1 if on_hand = 0, else 0 |

**Pivotable queries:**

```sql
-- Average on-hand by location, pivoted by quarter
SELECT
  l.site_name,
  AVG(CASE WHEN pr.calendar_quarter = 1 THEN f.on_hand_units END) AS q1_avg,
  AVG(CASE WHEN pr.calendar_quarter = 2 THEN f.on_hand_units END) AS q2_avg
FROM inventory_snapshot f
JOIN location l USING (location_key)
JOIN period pr USING (period_key)
WHERE pr.calendar_year = 2025
GROUP BY l.site_name;
```

---

### Domain 3: Finance General Ledger (grain: journal entry per day)

**Grain declaration:** One row per GL posting, per calendar day.

**Measures:**

| Measure | Type | Notes |
|---|---|---|
| `debit_amount_usd` | Additive | |
| `credit_amount_usd` | Additive | |
| `net_amount_usd` | Additive | debit - credit, stored for convenience |
| `budget_amount_usd` | Additive | From budget load, same grain |
| `variance_amount_usd` | Additive | actual - budget (components additive, store both) |

**Cross-domain join example, Sales + Finance:**

```sql
-- Revenue (Sales domain) vs. Recognized Revenue (Finance domain) by org, by month
SELECT
  o.org_name,
  pr.calendar_month,
  SUM(s.revenue_usd)                            AS sales_revenue,
  SUM(f.credit_amount_usd)                      AS gl_recognized_revenue,
  SUM(s.revenue_usd) - SUM(f.credit_amount_usd) AS recognition_gap
FROM org o
JOIN period pr ON pr.calendar_year = 2025
LEFT JOIN sales_orders s ON s.org_key = o.org_key AND s.period_key = pr.period_key
LEFT JOIN finance_gl f   ON f.org_key = o.org_key AND f.period_key = pr.period_key
                         AND f.account_type = 'REVENUE'
GROUP BY o.org_name, pr.calendar_month
ORDER BY o.org_name, pr.calendar_month;
```

This query works because `org` and `period` are conformed: the same keys join both datasets without transformation.

---

### Domain 4: HR Headcount (grain: employee-org per month end)

**Grain declaration:** One row per employee per org assignment, snapshot at last day of each calendar month.

**Measures:**

| Measure | Type | Notes |
|---|---|---|
| `headcount` | Semi-additive | Count of active employees at snapshot date |
| `fte` | Semi-additive | Full-time equivalent at snapshot date |
| `new_hires` | Additive | Employees whose start date falls in period |
| `terminations` | Additive | Employees whose end date falls in period |
| `salary_cost_usd` | Semi-additive | Monthly salary expense at snapshot |

---

### Domain 5: Marketing Spend (grain: campaign-channel-product per day)

**Grain declaration:** One row per campaign, per channel, per product, per calendar day.

**Measures:**

| Measure | Type | Notes |
|---|---|---|
| `spend_usd` | Additive | |
| `impressions` | Additive | |
| `clicks` | Additive | |
| `conversions` | Additive | |
| `cpc_usd` | Non-additive | `spend / clicks`. Compute at query time |
| `conversion_rate_pct` | Non-additive | `conversions / clicks`. Compute at query time |

---

## Cross-Domain Aggregation Patterns

### Pattern 1: Drill-Across (same conformed dimension, different domains)

Combine metrics from two fact tables by joining through shared conformed dimensions.

```sql
-- Cost per order: Marketing spend + Sales orders, by channel, by month
SELECT
  c.channel_name,
  pr.calendar_month,
  SUM(m.spend_usd)                                      AS marketing_spend,
  SUM(s.order_count)                                    AS orders,
  try_divide(SUM(m.spend_usd), SUM(s.order_count))     AS cost_per_order
FROM channel c
JOIN period pr ON pr.calendar_year = 2025
LEFT JOIN marketing_spend m ON m.channel_key = c.channel_key AND m.period_key = pr.period_key
LEFT JOIN sales_orders    s ON s.channel_key = c.channel_key AND s.period_key = pr.period_key
GROUP BY c.channel_name, pr.calendar_month;
```

### Pattern 2: Period-over-Period Comparison

Because period is conformed and hierarchical, YoY, MoM, and QoQ comparisons are structural, not special-case logic.

```sql
-- YoY revenue by product
SELECT
  p.product_name,
  SUM(CASE WHEN pr.calendar_year = 2024 THEN f.revenue_usd END) AS revenue_2024,
  SUM(CASE WHEN pr.calendar_year = 2025 THEN f.revenue_usd END) AS revenue_2025,
  try_divide(
    SUM(CASE WHEN pr.calendar_year = 2025 THEN f.revenue_usd END) -
    SUM(CASE WHEN pr.calendar_year = 2024 THEN f.revenue_usd END),
    SUM(CASE WHEN pr.calendar_year = 2024 THEN f.revenue_usd END)
  ) AS yoy_growth
FROM sales_orders f
JOIN product p  USING (product_key)
JOIN period pr  USING (period_key)
WHERE pr.calendar_year IN (2024, 2025)
GROUP BY p.product_name;
```

### Pattern 3: Forecasting Input Shape

ML forecasting libraries (Prophet, Nixtla, Statsforecast) expect a long/tidy format: entity identifier, period, measure. The conformed dataset produces this natively.

```sql
-- Forecasting-ready extract: revenue by product
SELECT
  p.product_id         AS entity_id,
  pr.calendar_date     AS ds,
  SUM(f.revenue_usd)   AS y
FROM sales_orders f
JOIN product p  USING (product_key)
JOIN period pr  USING (period_key)
WHERE pr.calendar_date >= DATEADD(month, -24, CURRENT_DATE)
GROUP BY p.product_id, pr.calendar_date
ORDER BY entity_id, ds;
```

Swap `product` for `location` or `org` to forecast any other dimension with zero structural change.

### Pattern 4: The Pivot Test

A well-structured dataset transforms cleanly from long (tidy) to wide (pivoted). The long form is what you store. The wide form is the acceptance test.

```
  LONG (stored)                             WIDE (pivoted)
  ────────────────────────────              ──────────────────────────────────
  product   │ month │ revenue               product   │  jan  │  feb  │  mar
  ──────────┼───────┼─────────              ──────────┼───────┼───────┼──────
  Widget A  │   1   │  1,000                Widget A  │ 1,000 │ 1,500 │ 1,200
  Widget A  │   2   │  1,500                Widget B  │   800 │ 1,100 │   950
  Widget A  │   3   │  1,200
  Widget B  │   1   │    800                    ▲           ▲               ▲
  Widget B  │   2   │  1,100              rows = entity  cols = period  values = measure
  Widget B  │   3   │    950
                  │
                  │  PIVOT (SUM(revenue) FOR month IN (1,2,3))
                  ▼
  one row per entity × one column per period × one additive measure
```

A dataset passes the pivot test when this query returns a valid result without transformation:

```sql
-- Standard pivot: entity × period → measure
SELECT *
FROM (
  SELECT
    p.product_name,
    pr.calendar_month,
    f.revenue_usd
  FROM sales_orders f
  JOIN product p USING (product_key)
  JOIN period pr USING (period_key)
  WHERE pr.calendar_year = 2025
)
PIVOT (
  SUM(revenue_usd)
  FOR calendar_month IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12)
);
```

> If this fails, one of three things is wrong: the measure is non-additive and must be pre-derived, the grain is mixed, or the period is not conformed.

---

## Design Checklist

Before publishing any dataset to the analytical layer, verify:

- [ ] Grain is declared in plain English on the dataset documentation
- [ ] Grain type is identified: transaction, periodic snapshot, or accumulating snapshot
- [ ] No two measures in the dataset imply different grains
- [ ] Every row has exactly one `period_key` (minimum); additional date keys named and documented
- [ ] All dimension foreign keys resolve to conformed dimension tables with matching surrogate keys
- [ ] All measures are at the declared grain: no implicit roll-ups stored
- [ ] Non-additive measures are documented as such; components (numerator, denominator) are stored separately
- [ ] Semi-additive measures are labeled with the correct aggregation behavior for the time axis
- [ ] The dataset can be pivoted: entity rows × period columns × one additive measure = valid result
- [ ] YoY and period-over-period queries work without CTEs or subquery hacks
- [ ] A drill-across join to at least one other domain dataset produces a correct result via shared conformed dimension keys
- [ ] Every business metric derived from this dataset has a named definition in the semantic/Gold layer
- [ ] Metric definitions reference measure column names explicitly, not other metrics
- [ ] Non-additive metrics expose numerator and denominator as separate queryable measures
- [ ] Metric version and effective date are recorded when definitions change

---

## Sharp Edges

**Grain mixing is silent and lethal.** Storing order-level and order-line-level data in the same fact table doubles aggregation silently. Declare once. Enforce strictly.

**Accumulating snapshots are updated, not appended.** This is the only fact table type where rows are modified after initial insert. ETL pipelines that treat accumulating snapshots as append-only will duplicate or miss milestone data.

**Conformed does not mean identical columns everywhere.** A conformed dimension shares key and core attributes. Domain-specific attributes can extend it without breaking conformity. The key and shared attributes must never diverge.

**Measure ≠ Metric. Conflating them breaks governance.** A measure is a column in a fact table. A metric is a named business calculation with a governed definition. Exposing measures directly to BI consumers without a metric layer guarantees definition drift across teams.

**Metric aggregation across dimensions is not always obvious.** Conversion rate at the channel level is not the average of product-level conversion rates. Composite metrics must specify their aggregation path explicitly. If they do not, consumers will compute them wrong and the numbers will appear to add up while being incorrect.

**Semi-additive measures require explicit handling at the semantic layer.** If your BI tool or semantic layer does not know a measure is semi-additive, it will SUM across time and produce wrong numbers. Document it. Enforce it in the metric definition.

**Non-additive measures should not be stored in fact tables.** Store the numerator and denominator as separate additive measures. Compute the ratio at query time or in a materialized view. Stored ratios lie when you aggregate the fact table to a different grain.

**Period without a declared fiscal calendar is incomplete.** If the business uses fiscal periods that differ from calendar, the period dimension must carry both. Queries that use fiscal attributes and calendar attributes interchangeably on the same fact table produce silent misalignment.

**Forecasting on semi-additive measures requires aggregation strategy declaration upfront.** Forecasting libraries do not know your measure is semi-additive. Feed them the correctly aggregated time series (average, not sum, across snapshots) or your forecasts will be wrong by construction.

---

## Summary

| Element | Role | Contract |
|---|---|---|
| **Period** | Time axis | Declared grain, conformed hierarchy, fiscal + calendar attributes, every fact table attaches here |
| **Grain** | Row contract | Plain-English declaration before schema design; type declared (transaction / snapshot / accumulating); never mixed within a table |
| **Dimensions** | Context axes | Conformed core set, atomic grain, surrogate keys, SCD strategy declared |
| **Measures** | Raw stored facts | Additive stored, non-additive computed, semi-additive labeled, all at declared grain |
| **Metrics** | Governed business calculations | Defined once in semantic layer, reference measures not other metrics, version-controlled, aggregation path explicit |

The framework is domain-agnostic. Sales, Finance, HR, Marketing, Operations, and Support can all publish fact tables into this contract and be queried, aggregated, compared, and forecasted with identical analytical patterns.

> Grain is the design contract. Measures are what you store. Metrics are what you govern. The pivot test is the acceptance criterion: necessary, not sufficient. A dataset that fails it is broken; a dataset that passes it still needs the reconciliation checks in its spec (row counts, control totals) before it is trusted.
