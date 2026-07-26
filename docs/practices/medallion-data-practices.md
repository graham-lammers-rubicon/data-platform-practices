# Medallion Architecture: Data Practices Reference

Databricks coined the term. The multi-hop layering concept is decades older. The name stuck, the pattern spread — Microsoft Fabric, Snowflake, Iceberg stacks all run it now. It belongs to the industry.

> **Bronze, Silver, Gold are data with receipts. Not folder names. Not a naming convention. A structural promise between data producers and every service that consumes it.**

| Layer | Quality | Consumers |
|---|---|---|
| **Bronze** | Raw | Silver pipelines only |
| **Silver** | Validated | Gold pipelines, data science |
| **Gold** | Governed, Trusted | Analytics, GenAI, APIs, MCP |

---

## Unity Catalog: The Governance Spine

Every layer lives inside Unity Catalog. That is not optional.

UC gives you fine-grained access control at the catalog, schema, table, row, and column level. It captures lineage automatically as data moves Bronze to Silver to Gold — queryable in `system.access.table_lineage`. Every read and write is audited in `system.access.audit`. No spreadsheet required. And no prefixes in table or column names — the schema tells you the layer, the COMMENT tells you what it is. Prefixes are a legacy habit from a world without metadata. That world is gone.

> **If any consuming service has a connection string pointing at Bronze, something has gone badly wrong.**

Recommended structure:

```
<env>_catalog
  ├── bronze    →  <entity>              (streaming tables, UC managed)
  ├── silver    →  <domain>              (fact tables, UC managed)
  └── gold      →  <domain>_<grain>      (materialized views or Delta, UC managed)
```

| Role | Bronze | Silver | Gold |
|---|---|---|---|
| Pipeline service principal | WRITE | WRITE | WRITE |
| Data engineers | READ | READ/WRITE | READ |
| Analysts / data scientists | ✗ | READ (approved) | READ |
| Downstream consuming services | ✗ | ✗ | READ |

---
---

# 🥉 Bronze — Capture Everything, Transform Nothing

> **Bronze is a crime scene. Land the data exactly as it arrived. Touch nothing. Every byte is evidence.**

Raw. Append-only. Schema-on-read. One table per source entity. No joins, no casting, no renaming, no business logic. Full stop.

Sources land here in whatever form they arrive — batch files, streaming events, or CDC feeds from operational systems. Databricks [recommends storing most fields](https://docs.databricks.com/aws/en/lakehouse/medallion) as `STRING`, `VARIANT`, or binary to survive unexpected schema changes. Add system columns on ingest. Always configure `rescuedDataColumn` — skip it and unknown fields vanish silently, and you'll find out six months later.

**System columns:**
```sql
_ingest_timestamp   TIMESTAMP   -- arrival time
_source_file        STRING      -- origin path or topic
_pipeline_id        STRING      -- Lakeflow run ID
_is_quarantined     BOOLEAN     -- failed validation
_raw_payload        STRING      -- rescue column
```

Bronze lands data in whatever shape the source sends it. Wide columns, compound codes, mixed types — all of it. That is intentional. The structure problem is Silver's to solve.

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
  _is_quarantined   BOOLEAN,
  _raw_payload      STRING
)
```

**Pattern (Lakeflow SDP):**
```sql
CREATE OR REFRESH STREAMING TABLE orders
COMMENT 'Raw OMS order events. Append-only. Source of truth for reprocessing.'
TBLPROPERTIES ('quality' = 'bronze')
AS SELECT
  *,
  current_timestamp()   AS _ingest_timestamp,
  _metadata.file_path   AS _source_file,
  false                 AS _is_quarantined
FROM STREAM read_files(
  '/Volumes/raw/landing/orders/',
  format            => 'json',
  schemaHints       => 'order_id STRING, order_date DATE, customer_id STRING',
  rescuedDataColumn => '_raw_payload'
);
```

## SCD Type 2 Belongs in Bronze

This is a genuine holy war. The conventional placement is Silver or Gold. Here's the case for Bronze.

The standard argument: history tracking is a "transformation," therefore Silver's job. That collapses the moment you ask — *what exactly is Silver transforming if Bronze didn't preserve the change?*

> **If Bronze drops the prior version of a record, the history is gone. You cannot reconstruct it from Silver. You cannot reconstruct it from Gold. It is gone.**

A dimension change arriving from source *is* source truth. The customer moved regions. The product was reclassified. That event happened at the source. Bronze is the system of record for source events — capturing it as a versioned row with `__START_AT` and `__END_AT` (as Lakeflow AUTO CDC produces) is faithful recording of what happened, not a business transformation.

Putting SCD2 in Silver also creates a responsibility collision: Silver now cleans *and* reconstructs history, with a fragile dependency on event ordering in streaming pipelines. Silver reads a versioned entity from Bronze and gets on with its actual job.

**Pattern — SCD2 in Bronze via AUTO CDC:**
```sql
CREATE OR REFRESH STREAMING TABLE customer
COMMENT 'Customer entity with full SCD2 history. Source-faithful. Versions preserved at ingest.'
TBLPROPERTIES ('quality' = 'bronze');

APPLY CHANGES INTO LIVE.customer
FROM STREAM read_files(
  '/Volumes/raw/landing/customer_cdc/',
  format            => 'json',
  rescuedDataColumn => '_raw_payload'
)
KEYS (customer_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY updated_at
COLUMNS * EXCEPT (operation, _raw_payload)
STORED AS SCD TYPE 2;
```

---
---

# 🥈 Silver — Validate, Clean, Conform

> **Silver is where raw data earns the right to be trusted.** Nothing leaves without being typed, deduplicated, and quality-checked. Do not write directly to Silver from source — bypass Bronze and you lose your audit trail and your ability to reprocess.

Cast types. Enforce naming standards. Handle nulls. Deduplicate. Apply conformed keys. Declare quality expectations with EXPECT constraints — bad rows get dropped or quarantined, not passed downstream. No business metric calculations. Silver stores raw measures at source grain and nothing more.

| Grain type | Lakeflow pattern | Behavior |
|---|---|---|
| Transaction | Streaming Table | Append on event |
| Periodic Snapshot | Streaming Table + MERGE | Insert every entity-period, even with no activity |
| Accumulating Snapshot | Streaming Table + AUTO CDC | Row updated as milestones complete |

**Pattern — transaction grain:**
```sql
CREATE OR REFRESH STREAMING TABLE orders (
  CONSTRAINT valid_revenue     EXPECT (revenue_usd >= 0)        ON VIOLATION DROP ROW,
  CONSTRAINT required_date     EXPECT (order_date IS NOT NULL)  ON VIOLATION FAIL UPDATE,
  CONSTRAINT required_customer EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW
)
COMMENT 'Cleaned order lines. One row per order line. Typed, validated, quarantine-filtered.'
TBLPROPERTIES ('quality' = 'silver')
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
FROM STREAM(prod_catalog.bronze.orders) o
WHERE o._is_quarantined = false;
```

Point-in-time balance measures (headcount, inventory, account balances) cannot be summed across time — label them semi-additive in the column COMMENT. Non-additive measures (rates, ratios) don't belong here at all; store numerator and denominator separately and compute the ratio in Gold.

Silver tables declare their grain in the COMMENT and store one variable per column, one observation per row. Wide source shapes are unpivoted here. Compound source codes are split into typed columns. The structure is long and atomic — ready to aggregate in any direction.

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
  shipped_quantity  BIGINT    COMMENT 'Additive — stored as component for fill_rate_pct in Gold'
)
```

---
---

# 🥇 Gold — Govern, Enrich, Serve

> **Gold is the governed semantic layer — the single trusted source for analytics pipelines, vector stores, GenAI retrieval indexes, REST APIs, and MCP servers alike. Every object has an owner, a definition, and a version. Design it once, for all of them. Without it, every consuming service invents its own truth.**

Materialized views are the default for frequently accessed aggregations. Pre-aggregated Delta tables when query performance demands it. Non-additive outputs must expose numerator and denominator as separate columns — do not make consumers reverse-engineer your math.

Gold tables are wide by design — intentionally shaped for consumption. Where Silver is long and atomic, Gold aggregates to a declared grain and exposes governed outputs. The structure must pivot cleanly: entity rows × period columns × one additive measure. Non-additive outputs are pre-computed at the declared grain and must not be re-aggregated.

```sql
-- Grain: one row per product per calendar month
-- Pivot test: product × month × revenue_usd must return valid result
-- Non-additive fill_rate_pct: components stored separately for verification
sales_by_product_month (
  product_name      STRING,
  calendar_year     INT,
  calendar_month    INT,                -- period at declared grain
  ordered_quantity  BIGINT    COMMENT 'Additive',
  revenue_usd       DECIMAL   COMMENT 'Additive',
  shipped_quantity_num BIGINT COMMENT 'Numerator for fill_rate_pct — verify independently',
  ordered_quantity_den BIGINT COMMENT 'Denominator for fill_rate_pct — verify independently',
  fill_rate_pct     DECIMAL   COMMENT 'Non-additive. Computed at this grain only. Do not re-aggregate.'
)
TBLPROPERTIES (
  'quality'        = 'gold',
  'metric.owner'   = 'revops',
  'metric.version' = '1'
)
```

**Pattern — materialized view:**
```sql
CREATE OR REFRESH MATERIALIZED VIEW sales_by_product_month
COMMENT 'Monthly revenue by product. Owner: RevOps. v1 2025-01-01.'
TBLPROPERTIES ('quality' = 'gold', 'metric.owner' = 'revops', 'metric.version' = '1')
AS SELECT
  p.product_name,
  pr.calendar_year,
  pr.calendar_month,
  SUM(s.ordered_quantity)                                        AS ordered_quantity,
  SUM(s.revenue_usd)                                            AS revenue_usd,
  SUM(s.shipped_quantity)                                       AS shipped_quantity_num,
  SUM(s.ordered_quantity)                                       AS ordered_quantity_den,
  SAFE_DIVIDE(SUM(s.shipped_quantity), SUM(s.ordered_quantity)) AS fill_rate_pct
FROM prod_catalog.silver.orders s
JOIN prod_catalog.bronze.customer c ON c.customer_id = s.customer_id AND c.__END_AT IS NULL
JOIN prod_catalog.silver.period  pr ON pr.calendar_date = s.order_date
GROUP BY p.product_name, pr.calendar_year, pr.calendar_month;
```

**Pattern — cross-domain join (define once, not per consumer):**
```sql
CREATE OR REFRESH MATERIALIZED VIEW customer_acquisition_cost
COMMENT 'CAC by channel per month. Owner: RevOps. Spans marketing_spend + headcount + orders.'
AS SELECT
  c.channel_name,
  pr.calendar_year,
  pr.calendar_month,
  SUM(m.spend_usd)                 AS marketing_spend_usd,
  SUM(h.salary_cost_usd)           AS sales_cost_usd,
  COUNT(DISTINCT s.entity_key)     AS new_customers,
  SAFE_DIVIDE(
    SUM(m.spend_usd) + SUM(h.salary_cost_usd),
    NULLIF(COUNT(DISTINCT s.entity_key), 0)
  )                                AS cac_usd
FROM prod_catalog.silver.channel c
JOIN prod_catalog.silver.period pr USING (period_key)
LEFT JOIN prod_catalog.silver.marketing_spend m ON m.channel_key = c.channel_key AND m.period_key = pr.period_key
LEFT JOIN prod_catalog.silver.headcount       h ON h.org_code = 'SALES' AND h.period_key = pr.period_key
LEFT JOIN prod_catalog.silver.orders          s ON s.channel_key = c.channel_key AND s.period_key = pr.period_key
                                       AND s.is_first_order = true
GROUP BY c.channel_name, pr.calendar_year, pr.calendar_month;
```

---
---

# Pipeline Architecture

One Lakeflow SDP pipeline per domain. It owns Bronze through Gold for that scope.

```
sales_pipeline
  ├── bronze.orders        (Auto Loader, streaming)
  ├── bronze.customer      (SCD2, AUTO CDC — history preserved at source layer)
  ├── silver.orders        (cleaned, typed)
  └── gold.sales_monthly   (materialized view)
```

Unqualified table names in a pipeline resolve to that pipeline's configured default catalog and schema. Reading from a table owned by a different pipeline requires a fully qualified three-part name (`catalog.schema.table`). This is why cross-layer reads in the patterns above use `prod_catalog.bronze.*` and `prod_catalog.silver.*` — not `LIVE.*`, which is legacy syntax no longer required in new pipelines.

Cross-domain Gold joins run in a separate pipeline or Lakeflow Job after upstream Silver is complete.

| Layer | Trigger | Pattern |
|---|---|---|
| Bronze | Continuous or triggered | Auto Loader, `read_files`, AUTO CDC |
| Silver | After Bronze | Streaming MERGE or type-cast reads |
| Gold | Scheduled or after Silver | Full refresh MV or incremental Delta |

---
---

# Checklist

### 🥉 Bronze
- [ ] One table per source entity, append-only, schema evolution on
- [ ] System columns present; `rescuedDataColumn` configured
- [ ] SCD2 entities tracked here via AUTO CDC with `STORED AS SCD TYPE 2`
- [ ] No prefixes in table or column names; intent lives in the COMMENT
- [ ] UC managed under `<env>_catalog.bronze`; pipeline principal WRITE only

### 🥈 Silver
- [ ] Types cast, nulls handled, deduplication applied
- [ ] EXPECT constraints declared with DROP or FAIL behavior
- [ ] Reads versioned entities from Bronze — does not reconstruct history
- [ ] Non-additive measures stored as components, not ratios
- [ ] Semi-additive measures labeled in column COMMENT
- [ ] UC managed under `<env>_catalog.silver`; lineage verifiable

### 🥇 Gold
- [ ] Every object has owner + version in TBLPROPERTIES or catalog tags
- [ ] Definitions reference Silver measures, not other Gold objects
- [ ] Non-additive outputs expose numerator and denominator separately
- [ ] Cross-domain joins defined here once
- [ ] All consuming services access Gold only; Silver/Bronze access revoked
- [ ] UC managed under `<env>_catalog.gold`; lineage verifiable

---

## Sharp Edges

| Risk | Mitigation |
|---|---|
| Schema evolution off on Bronze | Enable `MERGE SCHEMA`; configure `rescuedDataColumn` |
| Writing Silver directly from source | Always route through Bronze |
| SCD2 in Silver with streaming Bronze | Silver reconstructs history from ordering — fragile; move SCD2 to Bronze |
| Semi-additive measures SUMmed across time | Label in column COMMENT; enforce in consuming layer |
| Non-additive ratios stored in Silver | Store components; compute ratio at Gold |
| Non-additive outputs re-aggregated across dimensions | Expose numerator and denominator separately; document the constraint |
| Definitions living in consuming services | Single definition in Gold; revoke Silver/Bronze access |
| Writes outside Lakeflow | Lineage gaps; route everything through UC managed tables |

---

## Sources

| | |
|---|---|
| Medallion lakehouse architecture | https://docs.databricks.com/aws/en/lakehouse/medallion |
| Medallion architecture glossary | https://www.databricks.com/glossary/medallion-architecture |
| Data warehousing architecture | https://docs.databricks.com/aws/en/sql/get-started/data-warehousing-concepts |
| Lakehouse reference architectures | https://docs.databricks.com/aws/en/lakehouse-architecture/reference |
| Lakeflow Spark Declarative Pipelines | https://docs.databricks.com/aws/en/ldp/ |
| Unity Catalog | https://docs.databricks.com/aws/en/data-governance/unity-catalog/ |
| Unity Catalog system tables | https://docs.databricks.com/aws/en/administration-guide/system-tables/ |
