# Medallion Architecture: Data Practices Reference

Bronze, Silver, Gold is a structural promise between data producers and every service that consumes it. Not folder names. Not a naming convention.

| Layer | Quality | Consumers |
|---|---|---|
| **Bronze** | Raw | Silver pipelines only |
| **Silver** | Validated | Gold pipelines, data science |
| **Gold** | Governed, trusted | Analytics, GenAI, APIs, MCP |

---

## Unity Catalog: The Governance Spine

Every layer lives inside Unity Catalog. Not optional.

UC gives fine-grained access control at catalog, schema, table, row, and column level. Lineage is captured automatically and queryable in `system.access.table_lineage`. Reads and writes are audited in `system.access.audit`. No prefixes in table or column names: the schema tells you the layer, the COMMENT tells you what it is.

> **If any consuming service has a connection string pointing at Bronze, something has gone badly wrong.**

Structure:

```
<env>_catalog
  ├── bronze    →  <entity>              (streaming tables, UC managed)
  ├── silver    →  <domain>              (fact tables and entities, UC managed)
  └── gold      →  <domain>_<grain>      (materialized views or Delta, UC managed)
```

The access matrix is authoritatively defined in the [Access model](../governance/access-model.md). Summary:

| Role | Bronze | Silver | Gold |
|---|---|---|---|
| Pipeline service principal | READ/WRITE | READ/WRITE | READ/WRITE |
| Data engineers | dev: READ/WRITE; else READ | dev: READ/WRITE; else READ | dev: READ/WRITE; else READ |
| Analysts / data scientists | ✗ | READ (approved) | READ |
| Downstream consuming services | ✗ | ✗ | READ |

---
---

# 🥉 Bronze: Capture Everything, Transform Nothing

> **Land the data exactly as it arrived. Touch nothing.**

Raw. Append-only. Schema-on-read. One table per source entity. No joins, no casting, no renaming, no business logic.

Sources land here as batch files, streaming events, or CDC feeds. Databricks [recommends storing most fields](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/medallion) as `STRING`, `VARIANT`, or binary to survive schema changes. Always configure `rescuedDataColumn`; without it, fields that fail schema match are dropped silently.

**Prefer managed connectors.** Where [Lakeflow Connect](https://learn.microsoft.com/en-us/azure/databricks/ingestion/lakeflow-connect/) has a managed connector for the source (SaaS applications, database CDC), use it: the connector lands Bronze directly and removes the hand-built file-delivery layer. The `read_files` patterns below are for sources without a managed connector, where files land in the landing volume first.

**System columns:**
```sql
_ingest_timestamp   TIMESTAMP   -- arrival time
_source_file        STRING      -- origin path or topic
_rescued_data       STRING      -- rescue column: fields that failed schema match
```

`_rescued_data` holds only the fields that did not match the schema. It is not a full copy of the payload; the table row is the payload.

Wide columns, compound codes, mixed types all land as-is. Restructuring is Silver's job.

```sql
-- Example: source lands month columns as values, not a variable
-- Bronze preserves this exactly as received
orders (
  order_id        STRING,
  customer_code   STRING,    -- e.g. "US-ENT-0042" — compound, unparsed
  jan_revenue     STRING,    -- wide: month spread across columns
  feb_revenue     STRING,
  mar_revenue     STRING,
  _ingest_timestamp TIMESTAMP,
  _source_file      STRING,
  _rescued_data     STRING
)
```

**Pattern (Lakeflow SDP):**
```sql
CREATE OR REFRESH STREAMING TABLE bronze.orders
COMMENT 'Raw OMS order events. Append-only. Source of truth for reprocessing.'
AS SELECT
  *,
  current_timestamp()   AS _ingest_timestamp,
  _metadata.file_path   AS _source_file
FROM STREAM read_files(
  '${landing_root}/orders/',
  format            => 'json',
  schemaHints       => 'order_id STRING, order_date DATE, customer_id STRING',
  rescuedDataColumn => '_rescued_data'
);
```

`${landing_root}` is a pipeline configuration parameter set per bundle target (see [Pipeline architecture](#pipeline-architecture)). Never hardcode a catalog or environment in pipeline code.

## CDC feeds: land the events, apply them in Silver

A CDC feed lands in Bronze as an append-only event table: every insert, update, and delete event, with its operation and sequence columns, exactly as the source sent it. Do not apply the changes here. The event log is the history; any downstream state, including SCD2, can be rebuilt from it by replay.

```sql
CREATE OR REFRESH STREAMING TABLE bronze.customer
COMMENT 'Raw customer CDC events. Append-only. One row per change event. Replay source for silver.customer.'
AS SELECT
  *,
  current_timestamp()   AS _ingest_timestamp,
  _metadata.file_path   AS _source_file
FROM STREAM read_files(
  '${landing_root}/customer_cdc/',
  format            => 'json',
  rescuedDataColumn => '_rescued_data'
);
```

---
---

# 🥈 Silver: Validate, Clean, Conform

> **Silver is where raw data earns the right to be trusted.** Nothing leaves without being typed, deduplicated, and quality-checked. Never write to Silver directly from source; bypassing Bronze loses the audit trail and the ability to reprocess.

Cast types. Handle nulls. Deduplicate. Apply conformed keys. Declare quality expectations with EXPECT constraints; bad rows are dropped or fail the update, never passed downstream. No business metric calculations: Silver stores raw measures at source grain.

| Grain type | Lakeflow pattern | Behavior |
|---|---|---|
| Transaction | Streaming table | Append on event |
| Periodic snapshot | Materialized view | One row per entity-period, recomputed as periods close |
| Accumulating snapshot | AUTO CDC flow (SCD1) | Row updated as milestones complete |

**Pattern: transaction grain:**
```sql
CREATE OR REFRESH STREAMING TABLE silver.orders (
  CONSTRAINT valid_revenue     EXPECT (revenue_usd >= 0)        ON VIOLATION DROP ROW,
  CONSTRAINT required_date     EXPECT (order_date IS NOT NULL)  ON VIOLATION FAIL UPDATE,
  CONSTRAINT required_customer EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW
)
COMMENT 'Cleaned order lines. Grain: one row per order line per calendar day. Typed and validated.'
AS SELECT
  o.order_id,
  o.order_line_id,
  CAST(o.order_date  AS DATE)           AS order_date,
  CAST(o.customer_id AS STRING)         AS customer_id,
  CAST(o.sku         AS STRING)         AS product_sku,
  CAST(o.quantity    AS BIGINT)         AS ordered_quantity,
  CAST(o.revenue     AS DECIMAL(18,4))  AS revenue_usd,
  CAST(o.discount    AS DECIMAL(18,4))  AS discount_usd,
  CAST(o.shipped_qty AS BIGINT)         AS shipped_quantity
FROM STREAM(bronze.orders) o;
```

## SCD Type 2 lives in Silver

Bronze holds the append-only change events. Silver derives the versioned entity from them with Lakeflow AUTO CDC. This is the split that preserves both fidelity and rebuildability:

- The Bronze event table is the history. Nothing is lost when Silver is rebuilt; a full refresh replays the events and reproduces every version.
- Applying changes is a transformation: AUTO CDC sequences events, applies deletes, and drops columns. That work belongs in Silver, not in the layer whose contract is "transform nothing." An SCD2 table is also updated in place (end-dating), which breaks Bronze's append-only rule.

**Pattern: SCD2 via AUTO CDC (current syntax; `APPLY CHANGES INTO` and `LIVE.` are legacy):**
```sql
CREATE OR REFRESH STREAMING TABLE silver.customer
COMMENT 'Customer entity with full SCD2 history. Grain: one row per customer per version. Rebuildable from bronze.customer.';

CREATE FLOW customer_scd2 AS
AUTO CDC INTO silver.customer
FROM STREAM(bronze.customer)
KEYS (customer_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY updated_at
COLUMNS * EXCEPT (operation, _rescued_data, _ingest_timestamp, _source_file)
STORED AS SCD TYPE 2;
```

Current versions: `WHERE __END_AT IS NULL`. Point-in-time: `__START_AT <= t AND (__END_AT > t OR __END_AT IS NULL)`. The system columns excluded above remain queryable in `bronze.customer`.

Point-in-time balance measures (headcount, inventory, balances) cannot be summed across time; label them semi-additive in the column COMMENT. Ratios do not belong here at all: store numerator and denominator, compute the ratio in Gold.

Silver tables declare their grain in the COMMENT and should store one variable per column, one observation per row: long and atomic, ready to aggregate in any direction. Wide source shapes are unpivoted here; compound codes are split into typed columns. This is a strong recommendation, not a gate: when the source shape is volatile, a fixed unpivot silently drops new columns (they land in `_rescued_data` and go no further). For volatile sources, keep Silver closer to the source shape and monitor `_rescued_data` for non-empty rows; restructure when the shape stabilizes ([Tidy data](tidy-data.md): an analysis standard, not a storage mandate).

```sql
-- Grain declared: one row per order line per calendar day
-- Wide source (jan_revenue, feb_revenue...) unpivoted to long here
-- Compound customer_code split into typed columns
orders (
  order_id          STRING    COMMENT 'Source order identifier',
  order_line_id     STRING    COMMENT 'Source line identifier',
  order_date        DATE      COMMENT 'Transaction date — atomic grain for period join',
  period_key        INT       COMMENT 'FK → period conformed dimension',
  customer_segment  STRING    COMMENT 'Parsed from source customer_code — segment component',
  customer_region   STRING    COMMENT 'Parsed from source customer_code — region component',
  product_sku       STRING,
  ordered_quantity  BIGINT    COMMENT 'Additive — safe to SUM across all dimensions',
  revenue_usd       DECIMAL   COMMENT 'Additive — safe to SUM across all dimensions',
  discount_usd      DECIMAL   COMMENT 'Additive — safe to SUM across all dimensions',
  shipped_quantity  BIGINT    COMMENT 'Additive — component for fill_rate_pct in Gold'
)
```

---
---

# 🥇 Gold: Govern, Enrich, Serve

> **Gold is the governed semantic layer: the single trusted source for analytics, vector stores, GenAI retrieval, REST APIs, and MCP servers. Every object has an owner, a definition, and a version.**

Materialized views are the default for aggregations. Pre-aggregated Delta tables when query performance demands it. Governed business metrics are defined as [Unity Catalog metric views](../platform/genie-and-metric-views.md) over Gold and Silver objects: one YAML definition, safe re-aggregation, native Genie and dashboard integration.

Gold serving tables are wide by design, aggregated to a declared grain. The structure must pivot cleanly: entity rows × period columns × one additive measure. A non-additive output may be exposed at the declared grain only, with numerator and denominator as separate columns and a do-not-re-aggregate COMMENT. Gold reads Silver, never Bronze.

```sql
-- Grain: one row per product per calendar month
-- Pivot test: product × month × revenue_usd must return valid result
sales_product_monthly (
  product_sku       STRING,
  calendar_year     INT,
  calendar_month    INT,                -- period at declared grain
  ordered_quantity  BIGINT    COMMENT 'Additive',
  revenue_usd       DECIMAL   COMMENT 'Additive',
  shipped_quantity_num BIGINT COMMENT 'Numerator for fill_rate_pct',
  ordered_quantity_den BIGINT COMMENT 'Denominator for fill_rate_pct',
  fill_rate_pct     DECIMAL   COMMENT 'Non-additive. Computed at this grain only. Do not re-aggregate.'
)
TBLPROPERTIES (
  'quality'        = 'gold',
  'metric.owner'   = 'revops',
  'metric.version' = '1'
)
```

**Pattern: materialized view:**
```sql
CREATE OR REFRESH MATERIALIZED VIEW gold.sales_product_monthly
COMMENT 'Monthly sales by product. Grain: one row per product per calendar month. Owner: RevOps. v1 2025-01-01.'
TBLPROPERTIES ('quality' = 'gold', 'metric.owner' = 'revops', 'metric.version' = '1')
AS SELECT
  s.product_sku,
  pr.calendar_year,
  pr.calendar_month,
  SUM(s.ordered_quantity)                                       AS ordered_quantity,
  SUM(s.revenue_usd)                                            AS revenue_usd,
  SUM(s.shipped_quantity)                                       AS shipped_quantity_num,
  SUM(s.ordered_quantity)                                       AS ordered_quantity_den,
  try_divide(SUM(s.shipped_quantity), SUM(s.ordered_quantity))  AS fill_rate_pct
FROM silver.orders s
JOIN silver.period pr ON pr.calendar_date = s.order_date
GROUP BY s.product_sku, pr.calendar_year, pr.calendar_month;
```

**Pattern: cross-domain join (define once, not per consumer):**
```sql
CREATE OR REFRESH MATERIALIZED VIEW gold.customer_acquisition_cost
COMMENT 'CAC by channel per month. Grain: one row per channel per calendar month. Owner: RevOps.'
AS SELECT
  c.channel_name,
  pr.calendar_year,
  pr.calendar_month,
  SUM(m.spend_usd)                 AS marketing_spend_usd,
  SUM(h.salary_cost_usd)           AS sales_cost_usd,
  COUNT(DISTINCT s.entity_key)     AS new_customers,
  try_divide(
    SUM(m.spend_usd) + SUM(h.salary_cost_usd),
    COUNT(DISTINCT s.entity_key)
  )                                AS cac_usd
FROM silver.channel c
JOIN silver.period pr ON pr.calendar_year >= 2025
LEFT JOIN silver.marketing_spend m ON m.channel_key = c.channel_key AND m.period_key = pr.period_key
LEFT JOIN silver.headcount       h ON h.org_code = 'SALES' AND h.period_key = pr.period_key
LEFT JOIN silver.orders          s ON s.channel_key = c.channel_key AND s.period_key = pr.period_key
                                   AND s.is_first_order = true
GROUP BY c.channel_name, pr.calendar_year, pr.calendar_month;
```

---
---

# Pipeline Architecture

One Lakeflow SDP pipeline per domain. It owns Bronze through Gold for that scope.

```
sales-medallion (pipeline)
  ├── bronze.orders        (Auto Loader, streaming, append-only)
  ├── bronze.customer      (raw CDC events, append-only)
  ├── silver.orders        (cleaned, typed, EXPECT constraints)
  ├── silver.customer      (SCD2 via AUTO CDC)
  └── gold.sales_product_monthly   (materialized view)
```

Catalog and environment come from configuration, never from code:

- Table references in pipeline code are two-part `schema.table` names (`bronze.orders`, `silver.customer`). They resolve to the pipeline's default catalog, which the bundle target sets (`dev_catalog`, `nonprod_catalog`, `prod_catalog`). A three-part name with a hardcoded catalog breaks promotion; treat it as a defect.
- Environment-varying values (landing paths, lookback windows) are pipeline configuration parameters set per bundle target: `${param}` in SQL, `spark.conf.get("param")` in Python.
- `LIVE.` and `APPLY CHANGES INTO` are legacy syntax. Do not use them in new code.

Conformed dimensions (`silver.period`, `silver.entity`, `silver.product`, `silver.location`, `silver.channel`, `silver.org`) are built by one platform-owned pipeline, `conformed-dimensions`, not by domain pipelines. Cross-pipeline reads need no special syntax: `FROM silver.period` in a domain pipeline resolves to the same catalog the `conformed-dimensions` pipeline wrote to, because both pipelines get their default catalog from the same bundle target environment.

Cross-domain Gold joins (like `gold.customer_acquisition_cost`) run in a separate pipeline or Lakeflow Job, scheduled after upstream Silver completes, run as its own service principal (see [Access model](../governance/access-model.md)).

| Layer | Trigger | Pattern |
|---|---|---|
| Bronze | Continuous or triggered | Auto Loader, `read_files` |
| Silver | Same pipeline, after Bronze | Streaming tables, AUTO CDC flows |
| Gold | Scheduled or after Silver | Materialized view or incremental Delta |

Writes outside the pipeline (ad hoc `saveAsTable`, manual MERGE) bypass expectations and the declared contract. Route all writes through the domain pipeline.

---
---

# Checklist

### 🥉 Bronze
- [ ] One table per source entity, append-only, schema evolution on
- [ ] System columns present; `rescuedDataColumn => '_rescued_data'` configured
- [ ] CDC feeds land as append-only event tables; no changes applied in Bronze
- [ ] No prefixes in table or column names; intent lives in the COMMENT
- [ ] UC managed under `<env>_catalog.bronze`; grants per the access matrix

### 🥈 Silver
- [ ] Types cast, nulls handled, deduplication applied
- [ ] EXPECT constraints declared with DROP or FAIL behavior
- [ ] SCD2 entities derived here via `AUTO CDC ... STORED AS SCD TYPE 2`, rebuildable from Bronze events
- [ ] Non-additive measures stored as components, not ratios
- [ ] Semi-additive measures labeled in column COMMENT
- [ ] Grain declared in every table COMMENT
- [ ] UC managed under `<env>_catalog.silver`; lineage verifiable

### 🥇 Gold
- [ ] Every object has owner + version in TBLPROPERTIES or catalog tags
- [ ] Definitions reference Silver objects or a Gold serving table (metric views); never Bronze
- [ ] Business metrics defined as metric views ([Genie and metric views](../platform/genie-and-metric-views.md))
- [ ] Non-additive outputs expose numerator and denominator separately
- [ ] Cross-domain joins defined here once
- [ ] All consuming services access Gold only
- [ ] UC managed under `<env>_catalog.gold`; lineage verifiable

---

## Sharp Edges

| Risk | Mitigation |
|---|---|
| Schema evolution off on Bronze | Enable schema evolution; configure `rescuedDataColumn` |
| Writing Silver directly from source | Always route through Bronze |
| Applying CDC changes in Bronze | Bronze loses append-only fidelity and the replay source; land events raw, apply in Silver |
| Backfill through AUTO CDC | Replaying events out of order against an existing target corrupts version chains; full-refresh the SCD2 table so it rebuilds from Bronze |
| Late or out-of-order CDC events | `SEQUENCE BY` orders them, but only within what has arrived; verify the sequence column is reliable end-to-end |
| Semi-additive measures SUMmed across time | Label in column COMMENT; enforce in the metric view definition |
| Non-additive ratios stored in Silver | Store components; compute ratio at Gold or in a metric view |
| Non-additive outputs re-aggregated | Expose numerator and denominator; document the constraint |
| Definitions living in consuming services | Single definition in Gold; revoke lower-layer access |
| Hardcoded catalog in pipeline code | Two-part names plus bundle-target catalog; promotion redeploys the same code unchanged |
| Writes outside the pipeline | Bypass expectations and the contract; route through the domain pipeline |

---

## Sources

- Azure Databricks: [Medallion lakehouse architecture](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/medallion)
- Azure Databricks: [Lakeflow Spark Declarative Pipelines](https://learn.microsoft.com/en-us/azure/databricks/ldp/)
- Azure Databricks: [AUTO CDC and SCD Type 2](https://learn.microsoft.com/en-us/azure/databricks/ldp/cdc)
- Azure Databricks: [Pipeline parameters](https://learn.microsoft.com/en-us/azure/databricks/ldp/parameters)
- Azure Databricks: [Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/)
- Azure Databricks: [System tables](https://learn.microsoft.com/en-us/azure/databricks/admin/system-tables/)
- Azure Databricks: [Metric views](https://learn.microsoft.com/en-us/azure/databricks/uc-semantics/metric-views/)
