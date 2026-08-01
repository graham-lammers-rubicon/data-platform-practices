# Tidy Data: Core Concepts

**Source:** Wickham, H. (2014). "Tidy Data." *Journal of Statistical Software*, 59(10). https://doi.org/10.18637/jss.v059.i10

> **Status: optional practice, low priority.** This doc is the theory behind the recommended layer shapes (Silver long, Gold wide), not a platform gate. The reshaping is a hard transition for people and tooling; treat it as a nice-to-have and adopt it per domain when capacity allows. Mandatory regardless of stored shape: grain declared in the COMMENT, typed columns, additivity labeled, and the pivot test on published datasets.

---

## What It Is

Tidy data is a standard way to structure tabular datasets so that manipulation, modeling, and visualization require minimal reshaping. The structure links physical layout to semantics: the shape of the table reflects what the data means.

The framework was formalized by Hadley Wickham in 2014 and underpins the R tidyverse ecosystem. The same principles apply in Python (pandas), SQL, and any columnar data store.

---

## The Three Rules

| Rule | What it means |
|---|---|
| Each variable is a column | One and only one variable per column |
| Each observation is a row | One and only one observation per row |
| Each type of observational unit is its own table | Don't mix granularities in a single table |

Violating any of these rules creates friction: you must reshape the data before you can compute on it.

---

## Common Violations (Messy Patterns)

### 1. Column headers are values, not variable names

Wide format where time periods, categories, or measurements spread across columns.

```
# Messy (wide)
country   | 1999  | 2000
----------|-------|------
Brazil    | 37737 | 80488
China     | 212258| 213766

# Tidy (long)
country   | year  | cases
----------|-------|------
Brazil    | 1999  | 37737
Brazil    | 2000  | 80488
China     | 1999  | 212258
China     | 2000  | 213766
```

**Fix:** pivot longer (melt / unpivot).

### 2. Multiple variables stored in one column

A single column encodes more than one variable, often via string concatenation or codes.

```
# Messy
column: "m014" encodes sex=male, age_group=0-14

# Tidy
sex | age_group | n
----|-----------|---
m   | 014       | 0
```

**Fix:** split / parse the column into separate variables.

### 3. Variables stored in both rows and columns

Some variables appear as column headers, others as row values in the same table.

```
# Messy
id      | element | d1   | d2   | ...
--------|---------|------|------|
MX17004 | tmax    | null | 27.3 |
MX17004 | tmin    | null | 14.4 |

# Tidy
id      | day | tmax | tmin
--------|-----|------|-----
MX17004 | 2   | 27.3 | 14.4
```

**Fix:** pivot longer to collapse day columns, then pivot wider to spread element rows into columns.

### 4. Multiple observational units in one table

Mixing different granularities (e.g., song metadata mixed with weekly chart position) duplicates data and creates update anomalies.

**Fix:** normalize into separate tables joined by a key. This aligns with relational database 3NF.

### 5. One observational unit spread across multiple tables / files

The same entity's data is split across files by year, region, or cohort, with no consistent schema.

**Fix:** bind / union the files into one table with a discriminator column.

---

## Wide vs. Long: When to Use Each

| Format | Use for |
|--------|---------|
| Long (tidy) | Analysis, filtering, grouping, modeling, visualization, column-store databases |
| Wide | Human-readable reporting, pivot tables, matrix operations, certain ML input formats, row-store databases |

Wide is how humans naturally record data. Long is what computation tools expect. The skill is moving between them efficiently.

---

## Reshaping Operations

| Operation | Direction | R | Python (pandas) | SQL |
|-----------|-----------|---|-----------------|-----|
| Melt / unpivot | Wide → Long | `pivot_longer()` | `melt()` | `UNPIVOT` |
| Pivot | Long → Wide | `pivot_wider()` | `pivot()` / `pivot_table()` | `PIVOT` |
| Split column | One → Many | `separate()` | `str.split()` | string functions |
| Unite columns | Many → One | `unite()` | `str.cat()` | `CONCAT` |

---

## Why It Matters for Platforms

Tidy structure is not just a coding convention. It is a contract between data producers and consumers.

- **Tooling assumes it.** Most ML libraries, visualization tools, and SQL aggregations expect one variable per column.
- **Pipelines break on wide data.** Dynamic column counts make SQL, Spark, and streaming schemas brittle.
- **Lakehouse / bronze-silver-gold maps directly.** Bronze often lands in messy wide form. Silver is where you tidy it. Gold is where you may intentionally widen it again for reporting.
- **Tidy raw tables, wide materialized views.** Keep facts tidy for auditability and extension. Widen only at the presentation layer.

---

## Limits and Sharp Edges

- Tidy is not always optimal for storage. Wide sparse tables use less space in row stores; tidy long tables are better for column stores.
- Performance is context-dependent. Benchmark before assuming tidy is faster.
- "What counts as a variable" is ambiguous in some domains. The definition of observation shifts with analysis goals.
- Tidy data is a standard for analysis, not a replacement for normalization. Production databases often need proper 3NF with foreign keys.
- Tidy tools create a closed ecosystem. Tools built for tidy data do not automatically support other valid data structures.

---

## Reference

- Original paper: https://vita.had.co.nz/papers/tidy-data.pdf
- R for Data Science (tidy chapter): https://r4ds.hadley.nz/data-tidy.html
- tidyr package docs: https://tidyr.tidyverse.org/
