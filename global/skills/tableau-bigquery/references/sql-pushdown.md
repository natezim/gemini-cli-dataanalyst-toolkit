# SQL Push-Down: How Tableau Generates SQL for BigQuery

## Table of Contents
1. [How to view the SQL Tableau is sending](#viewing-sql)
2. [Query Fusion](#query-fusion)
3. [Custom SQL behavior and the wrapping problem](#custom-sql)
4. [LOD expressions → nested subquery JOINs](#lod-expressions)
5. [COUNTD behavior](#countd)
6. [Table calculations: never pushed down](#table-calculations)
7. [Context filters and their effect on subqueries](#context-filters)

---

## 1. Viewing the SQL Tableau sends {#viewing-sql}

Three methods:

**Performance Recording** (Tableau Desktop): `Help → Settings & Performance → Start Performance Recording` → interact with dashboard → Stop → opens recording workbook showing SQL per event.

**Tableau log files**: `Documents/My Tableau Repository/Logs/log.txt` → search for `end-query`

**BigQuery Query History** (most complete): Cloud Console → BigQuery → Job History. Filter by label `tableau` if your org sets job labels. Shows bytes scanned, execution plan, duration.

---

## 2. Query Fusion {#query-fusion}

**What it is:** Since Tableau 9.0, worksheets querying at the same granularity on the same data source are merged into a single SQL query. Observed ~2× speedup in benchmark testing.

**How to confirm it's working:** In Performance Recording, fused queries appear as a single "Executing Query" event for a `Null` worksheet (not a specific sheet name).

**How to encourage it:**
- Align worksheet granularity (same dimensions) across dashboard sheets
- Use the same data source for related sheets
- Live connections only (extracts don't benefit)

**What breaks it:**
- Different levels of detail across sheets
- Custom SQL data sources (fusion is disabled)
- LOD expressions that change the query grain

---

## 3. Custom SQL: the wrapping problem {#custom-sql}

### The fundamental behavior

Tableau **always** wraps Custom SQL in an outer subquery:

```sql
-- What you write:
SELECT customer_id, order_date, order_total
FROM `project.dataset.orders`
WHERE order_date >= '2024-01-01'

-- What Tableau sends to BigQuery:
SELECT `Custom SQL Query`.`customer_id`,
       SUM(`Custom SQL Query`.`order_total`) AS `sum_order_total`
FROM (
    SELECT customer_id, order_date, order_total
    FROM `project.dataset.orders`
    WHERE order_date >= '2024-01-01'
) `Custom SQL Query`
WHERE `Custom SQL Query`.`region` = 'West'   -- Tableau-applied shelf filter
GROUP BY 1
```

### What this breaks

| Optimization | Native tables | Custom SQL |
|-------------|--------------|------------|
| Join culling | ✅ | ❌ |
| Query fusion | ✅ | ❌ |
| Relationship-model filtering | ✅ | ❌ |
| Partition pruning (with outer filter) | Usually ✅ | Sometimes ❌ |

**Join culling** is the biggest loss: if your Custom SQL joins orders + customers + products,
every sheet interaction executes the full 3-table join even if only 1 column from `customers`
is used. With native tables, Tableau would query only the tables needed.

### CTEs work, but aren't materialized

BigQuery supports `WITH` clauses inside Custom SQL subqueries. However, BigQuery does **not**
materialize non-recursive CTEs — each reference re-executes the CTE definition.

```sql
-- This works in Tableau Custom SQL on BigQuery:
WITH CustomerFirstOrder AS (
    SELECT customer_id, MIN(order_date) AS first_order_date
    FROM `project.dataset.orders`
    GROUP BY customer_id
)
SELECT o.customer_id, o.order_total
FROM `project.dataset.orders` o
JOIN CustomerFirstOrder f
  ON o.customer_id = f.customer_id
  AND o.order_date = f.first_order_date
```

Multiple references to `CustomerFirstOrder` = multiple scans of the base table.

### Parameterized Custom SQL for guaranteed partition pruning

Tableau parameters use `<Parameters.ParamName>` syntax and are substituted as literal values
before the SQL reaches BigQuery:

```sql
-- Custom SQL with Tableau parameter:
SELECT order_id, order_date, customer_id, order_total
FROM `project.dataset.orders`
WHERE order_date >= <Parameters.Start Date>
  AND order_date <= <Parameters.End Date>

-- Tableau sends (with user-selected dates):
WHERE order_date >= DATE '2024-01-01' AND order_date <= DATE '2024-12-31'
```

This generates constant-expression predicates **inside** the subquery, where BigQuery's
partition pruning can act on them directly. This is more reliable than relying on outer
WHERE clause pushthrough.

### Custom SQL rules for BigQuery

- All columns must be aliased (unnamed aggregations error when wrapped)
- No DDL statements
- Single SELECT only (no semicolons, no multi-statement)
- No top-level ORDER BY (Tableau re-sorts anyway)
- Use Standard SQL mode (Legacy SQL mode breaks CTEs and LODs)

### The recommended alternative: BigQuery Views

```sql
-- Instead of Custom SQL in Tableau:
CREATE OR REPLACE VIEW `project.dataset.v_daily_sales` AS
SELECT
    DATE(order_timestamp) AS order_date,
    region,
    product_category,
    SUM(revenue) AS total_revenue,
    COUNT(*) AS order_count
FROM `project.dataset.raw_orders`
GROUP BY 1, 2, 3;
```

Connect natively to `v_daily_sales` — all optimizations re-enable, and the view definition
is invisible to end users.

---

## 4. LOD expressions → nested subquery JOINs {#lod-expressions}

LOD expressions are **pushed down to BigQuery as SQL**, not computed in Tableau's engine.
Tableau generates a main query for view-level dimensions and JOINs it with subqueries for
each LOD. Example:

```sql
-- Calculation: {FIXED [State] : COUNTD([Order ID])}
-- On a view showing Country, State:

SELECT `t0`.`Country`, `t0`.`State`,
       `t1`.`__measure__0` AS `fixed_state_orders`
FROM (
    SELECT `Country`, `State` FROM `project.dataset.orders` GROUP BY 1, 2
) `t0`
INNER JOIN (
    SELECT `State`, COUNT(DISTINCT `Order_ID`) AS `__measure__0`
    FROM `project.dataset.orders` GROUP BY 1
) `t1`
ON `t0`.`State` IS NOT DISTINCT FROM `t1`.`State`
```

**Key implications:**
- Each LOD adds a subquery + JOIN — BigQuery may scan the base table multiple times
- Three FIXED LODs on a 500M-row table = 4+ subqueries, 3 JOINs, each independently scanning
- LOD expressions **ignore** dimension filters unless those filters are added to context
- `IS NOT DISTINCT FROM` is used for NULL-safe join comparisons

**When to pre-compute LODs in materialized views:**
- Same LOD appears in 3+ workbooks or 5+ sheets
- Base table > 100M rows
- LOD uses COUNT(DISTINCT) on high-cardinality columns

### LOD filter behavior (critical)

| Filter type | FIXED LOD respects it? | INCLUDE/EXCLUDE LOD? |
|------------|----------------------|---------------------|
| Data source filter | ✅ Always | ✅ Always |
| Context filter | ✅ Yes | ✅ Yes |
| Regular dimension filter | ❌ No (FIXED ignores dimension filters) | ✅ Yes |
| Table calculation filter | ❌ No | ❌ No |

To make a FIXED LOD respect a dimension filter → add that filter to context.

---

## 5. COUNTD behavior {#countd}

**Standard SQL mode (default since Tableau 10.1):** `COUNT(DISTINCT col)` is **exact**.
The old Legacy SQL approximation issue is gone.

```sql
-- Tableau generates:
SELECT COUNT(DISTINCT `t0`.`Customer_ID`) AS `ctd:Customer_ID:ok`
FROM `project.dataset.orders` `t0`
```

**For approximate counts (manual):** Use a calculated field with RAWSQL:
```
RAWSQLAGG_INT("APPROX_COUNT_DISTINCT(%1)", [Customer_ID])
```

`APPROX_COUNT_DISTINCT` is dramatically faster on large cardinalities and typically within 1%
of exact. Use when exact counts aren't required (dashboards showing "~10.4M users").

---

## 6. Table calculations: never pushed down {#table-calculations}

Table calculations (RUNNING_SUM, RANK, WINDOW_AVG, INDEX, LOOKUP, PREVIOUS_VALUE) are
**always** computed in Tableau's in-memory engine after results return from BigQuery.

**In live mode:** Tableau executes SQL → fetches all rows needed for the calculation →
computes in memory. Every interaction re-fetches from BigQuery.

**In extract mode:** Same calculation runs on the local Hyper engine — faster and no BigQuery charges.

**The problem:** A table calculation needing 100K aggregated rows transfers all 100K from
BigQuery on every dashboard interaction.

**Mitigation strategies:**
1. Replace with LOD expressions where functionally equivalent
2. Pre-aggregate in a BigQuery materialized view to reduce row count
3. Convert to extract for dashboards dominated by table calculations
4. Use RAWSQL window functions if BigQuery-side computation is acceptable

---

## 7. Context filters {#context-filters}

**Without context filter:** A dimension filter (e.g., `Region = 'Europe'`) appears only on
the outer query. FIXED LODs ignore it — they compute across all regions.

**With context filter:** The filter condition embeds in **every** subquery, including LOD
subqueries. This enables partition pruning in each sub-scan.

```sql
-- Region = 'Europe' as CONTEXT filter:
INNER JOIN (
    SELECT Region, SUM(Sales) AS fixed_sales
    FROM orders
    WHERE Region = 'Europe'   -- ← embedded in LOD subquery
    GROUP BY 1
) t1
```

**Important BigQuery distinction:** Unlike SQL Server/MySQL, BigQuery does NOT create
temporary tables for context filters. Context filters become embedded WHERE clauses in
subqueries. No "query once, reuse" benefit — each subquery runs independently.

**When to use context filters on BigQuery:**
- ✅ FIXED LODs need to respect a user-applied filter
- ✅ Top N filters should operate within a filtered scope
- ✅ Filter is on a partitioned/clustered column with high selectivity (>90% filtered)
- ❌ Filter doesn't significantly reduce data volume
- ❌ Column is not partitioned or clustered
- ❌ You're using INCLUDE/EXCLUDE LODs (they already respect dimension filters)

Reference: https://help.tableau.com/current/pro/desktop/en-us/calculations_calculatedfields_lod_filters.htm
