# Cost Control: Partitions, BI Engine, Materialized Views, Query Governors

## Table of Contents
1. [Partition filter enforcement](#partitions)
2. [Data source filters](#datasource-filters)
3. [BI Engine](#bi-engine)
4. [Materialized views](#materialized-views)
5. [Query governors](#governors)
6. [Cost formula and estimation](#cost)

---

## 1. Partition filter enforcement {#partitions}

### Server-side: `require_partition_filter = true`

The single most effective cost control for Tableau + BigQuery. Rejects any query lacking a
partition filter **before scanning any data** — zero bytes processed, zero cost.

```sql
ALTER TABLE `my_project.my_dataset.events`
SET OPTIONS (require_partition_filter = true);
```

Error returned when filter is missing:
`Cannot query over table without a filter over column(s) that can be used for partition elimination`

**The Tableau metadata query problem:** When connecting to a table with this option enabled,
Tableau's initialization queries (schema detection, dimension value lookups) may lack
partition filters, causing connection errors.

**Workarounds:**

Option A — BigQuery View (recommended):
```sql
CREATE VIEW `project.dataset.v_recent_events` AS
SELECT * FROM `project.dataset.events`
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY);
```
Connect Tableau to the view. View doesn't have `require_partition_filter`, so metadata
queries succeed. All queries against the view automatically include the date filter.

Option B — Custom SQL with Tableau parameter (see `sql-pushdown.md`)

Option C — Add a data source filter on the partition column immediately after connecting

### What actually triggers partition pruning

Only **constant expressions** enable pruning at query planning time:

```sql
-- ✅ PRUNES: constant literal
WHERE event_date = '2026-01-01'

-- ✅ PRUNES: constant function
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)

-- ❌ DOES NOT PRUNE: subquery
WHERE event_date = (SELECT MAX(event_date) FROM other_table)

-- ❌ DOES NOT PRUNE: function on partition column
WHERE EXTRACT(YEAR FROM event_date) = 2024

-- ❌ DOES NOT PRUNE: LIKE or REGEXP
WHERE CAST(event_date AS STRING) LIKE '2024%'
```

**Tableau-specific:** Discrete (blue pill) date parts often generate `EXTRACT(YEAR ...)` —
these defeat pruning. Always use **continuous (green pill) date filters** on partition columns.
They generate `>=` / `<=` comparisons that prune correctly.

---

## 2. Data source filters {#datasource-filters}

Data source filters are applied to **all queries** from a published data source, are invisible
to end users, and generate WHERE clauses on every query Tableau sends.

**Setup:** Data Source page → Add Filter → select partition column → set relative date range.

This generates on every query:
```sql
WHERE `event_date` >= DATE '2026-01-02' AND `event_date` <= DATE '2026-04-02'
```

### How data source filters interact with Custom SQL

Data source filters land on the **outer** wrapper query, not inside your Custom SQL:

```sql
-- Your Custom SQL:
SELECT * FROM `project.dataset.events`

-- Tableau sends:
SELECT ... FROM (SELECT * FROM `project.dataset.events`) t
WHERE t.event_date >= DATE '2026-01-02'   -- outer WHERE
```

BigQuery's optimizer **can** push simple predicates through a trivial `SELECT *` subquery.
For complex Custom SQL (aggregations, CTEs, multi-table joins), the optimizer **may not**
push the filter through — causing a full scan inside the subquery.

**Rule:** Embed partition filters inside Custom SQL using parameters. Use data source filters
as a backup, not the primary mechanism.

### Tableau 2025.1 per-table data source filter scoping

New in 2025.1: Data source filters have a scope setting:
- `<Table> and all related tables` — legacy default, filter applies across all joined tables
- `<Table> only` — per-table scoping, filter applies only to that table's query

Use per-table scoping to ensure a partition filter on a large fact table doesn't unnecessarily
filter joined dimension tables.

---

## 3. BI Engine {#bi-engine}

### What BI Engine accelerates

BI Engine is an in-memory query acceleration service that caches frequently accessed data
and accelerates SQL queries transparently. It works with Tableau live connections — no
Tableau-side configuration required.

**Accelerated:** Standard SQL patterns that Tableau generates — SELECT, GROUP BY, WHERE,
ORDER BY, INNER JOIN, LEFT OUTER JOIN (up to 4 dimension tables).

**Not accelerated (falls back to standard BigQuery):**
- Analytic functions: PERCENTILE_CONT, PERCENTILE_DISC
- JavaScript UDFs
- JOINs involving more than 4 dimension tables
- Some complex window functions
- Queries using tables NOT in the preferred tables list

### Setup

Create a BI Engine reservation in the same project and region as your data:

```sql
-- Via Cloud Console: BigQuery → BI Engine → Create Reservation
-- Or via API/terraform. Key parameters:
--   size_gb: amount of RAM to reserve (start with 10-50 GB)
--   preferred_tables: list of specific tables to cache (optional but recommended)

ALTER BI_CAPACITY `my-project.region-us.default`
SET OPTIONS (
    size_gb = 10,
    preferred_tables = [
        'my-project.my_dataset.fact_sales',
        'my-project.my_dataset.dim_product',
        'my-project.my_dataset.dim_customer'
    ]
);
```

**Critical:** The billing project used in Tableau's BigQuery connection must match the project
where the BI Engine reservation exists. A project/region mismatch silently prevents acceleration.

### Right-sizing BI Engine

BI Engine caches only **frequently accessed columns and rows**, not entire tables. You do
not need GB equal to your total table size.

Target: **>80% full acceleration rate** monitored via INFORMATION_SCHEMA (see `information-schema.md`).

Starting point: size ≈ working set of commonly queried partitions × compression factor (~0.3).
Example: 100 GB table, queries typically touch last 30 days of a date-partitioned table with
10 GB/month → working set ≈ 10 GB → start with 3-5 GB reservation.

Maximum default reservation: **250 GB** (requestable increase via support).

**Requirements:**
- Enterprise or Enterprise Plus edition required
- Views cannot be in preferred tables list (base tables only)
- For materialized views: both the MV and its base tables must be listed
- For JOIN queries: all tables in the JOIN must be in the preferred list

---

## 4. Materialized views {#materialized-views}

### Incremental MVs with smart tuning (recommended)

BigQuery automatically rewrites queries against base tables to use eligible materialized views.
Tableau queries `fact_orders` → BigQuery internally reads from `mv_daily_sales`. **No changes
to Tableau workbooks needed.**

```sql
CREATE MATERIALIZED VIEW `project.dataset.mv_daily_sales`
OPTIONS (
    enable_refresh = true,
    refresh_interval_minutes = 30
)
AS SELECT
    DATE(order_timestamp) AS order_date,
    region,
    product_category,
    COUNT(*) AS order_count,
    SUM(revenue) AS total_revenue
FROM `project.dataset.fact_orders`
GROUP BY 1, 2, 3;
```

Smart tuning conditions (all must be met):
- MV must be incremental (GROUP BY without OUTER JOIN, window functions, or HAVING)
- MV belongs to same project as base table
- Query uses the same base tables as the MV
- All rows and columns needed by the query are in the MV
- MV is within `max_staleness` window (if set)

### Non-incremental MVs for complex SQL

Use `allow_non_incremental_definition = true` for OUTER JOINs, UNION ALL, analytic functions,
HAVING. These require `max_staleness` and perform full refreshes. No smart tuning — Tableau
must connect directly to the MV.

```sql
CREATE MATERIALIZED VIEW `project.dataset.mv_sales_enriched`
OPTIONS (
    enable_refresh = true,
    refresh_interval_minutes = 60,
    max_staleness = INTERVAL "4:0:0" HOUR TO SECOND,
    allow_non_incremental_definition = true
)
AS SELECT
    o.order_date,
    o.revenue,
    c.customer_segment,
    p.category
FROM `project.dataset.orders` o
LEFT JOIN `project.dataset.customers` c ON o.customer_id = c.id
LEFT JOIN `project.dataset.products` p ON o.product_id = p.id;
```

### Connecting Tableau to a materialized view

MVs appear as regular tables in Tableau's connection UI. Drag the MV onto the canvas and
use a live connection. No special configuration needed. The `max_staleness` option means
BigQuery may serve from the MV without reading base tables if data is within the staleness window.

### Cost savings example

200 Tableau queries/day scanning 10 GB each from a 500 GB base table:

| Scenario | Daily scan | Monthly cost |
|----------|-----------|-------------|
| No MV (base table) | 2 TB/day | ~$375/month |
| MV pre-aggregated to 500 MB | 100 GB/day | ~$18.75/month |

### Key limitations

- Max **20 MVs per base table**
- Incremental MVs require the largest/most-changing table as the **leftmost** in JOINs
- DML (UPDATE/MERGE/DELETE) on base tables invalidates incremental updates until next full refresh
- Must refresh within **3 days** (streaming buffer retention limit)
- April 2024: non-incremental MVs now support cross-dataset references (new)

---

## 5. Query governors {#governors}

### Tableau Server time-based limits

```bash
# View rendering timeout (default: 1800s = 30 min):
tsm configuration set -k vizqlserver.querylimit -v 180     # 3 minutes recommended
tsm pending-changes apply

# Extract refresh timeout (default: 7200s = 2 hours):
tsm configuration set -k backgrounder.querylimit -v 9000
tsm pending-changes apply
```

**Limitation:** Time-based only. A fast query scanning 5 TB in 10 seconds costs $28 but
passes the time limit. Cannot set byte limits natively in Tableau.

### BigQuery per-query byte limit (API only)

`maximum_bytes_billed` rejects queries exceeding a byte threshold before scanning — zero cost.
Not exposed in Tableau's native connector. Only accessible via:
- BigQuery API/SDK directly
- Custom JDBC connection string
- Third-party connectors (e.g., CData exposes `MaximumBytesBilled`)

```python
# Python example (for reference — not usable directly from Tableau):
job_config = bigquery.QueryJobConfig(
    maximum_bytes_billed=100 * 1024 ** 3  # 100 GB hard cap
)
```

### BigQuery custom daily quotas (recommended primary control)

Most effective budget guardrail. Set at project level and per-user level. Resets midnight Pacific.

```
IAM & Admin → Quotas & System Limits → BigQuery API
→ "Query usage per day" → Edit → e.g., 10 TiB per day
→ "Query usage per day per user" → Edit → e.g., 1 TiB per day
```

**Note (September 2025+):** New projects default to 200 TiB/day limit. Existing "unlimited"
projects receive a custom limit based on historical peak usage.

### Slot reservations for predictable cost

Assign Tableau's project to a reservation with fixed slots — converts per-TB to flat capacity cost.

| Edition | Price | 100 slots cost |
|---------|-------|---------------|
| Standard | $0.04/slot-hour | $96/day |
| Enterprise | $0.06/slot-hour | $144/day |
| Enterprise Plus | $0.10/slot-hour | $240/day |

1-year commitment: ~20% discount. 3-year: ~40%.
Benefit: Tableau workloads are isolated and compute cost is capped regardless of bytes scanned.

---

## 6. Cost formula and estimation {#cost}

### On-demand pricing

```
Cost ($) = (total_bytes_billed / 1,099,511,627,776) × $6.25
```

Note: BigQuery prices in **TiB** (tebibytes = 1024⁴ bytes), not TB (1000⁴ bytes).

Key rules:
- First **1 TiB/month free** per billing account
- Minimum charge: **10 MB per table referenced** per query
- Minimum charge: **10 MB per query**
- LIMIT clauses do NOT reduce bytes scanned (exception: clustered tables)
- Cached results: **free** (same query, same data, within 24 hours)
- Error queries: **free** (if error occurs before scan begins)

### Storage pricing

| Storage type | Price |
|-------------|-------|
| Active (< 90 days unchanged) | $0.02/GiB/month (logical) |
| Long-term (≥ 90 days unchanged) | $0.01/GiB/month (automatic) |
| Physical (compressed) | ~50-80% of logical price |

Reference: https://cloud.google.com/bigquery/pricing
