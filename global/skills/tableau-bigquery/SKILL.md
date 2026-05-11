---
name: tableau-bigquery
description: >
  Expert guidance for Tableau + BigQuery live connection workflows. Use this skill
  whenever the user is working on Tableau connected to BigQuery — this includes:
  troubleshooting slow dashboards, optimizing live connection queries, setting up
  partition filters, diagnosing high BigQuery costs from Tableau, configuring Custom SQL,
  understanding LOD/COUNTD behavior, setting up BI Engine, creating materialized views
  for Tableau, managing service account vs OAuth auth, query governor settings, or
  reviewing INFORMATION_SCHEMA queries for cost/slot monitoring. Also use when the
  user asks about Tableau performance on large tables, Tableau SQL push-down, or how
  Tableau generates SQL against BigQuery.
---

# Tableau + BigQuery Live Connections

Expert skill for production Tableau + BigQuery workflows. Covers SQL push-down behavior,
cost control, auth, and monitoring. Research sourced from Tableau 2025 and BigQuery 2024-2025 docs.

## The single most important mental model

**Tableau always wraps your data in an outer query.** Every filter, calculation, and LOD
expression lands on top of a subquery. Understanding what gets pushed down vs. computed
client-side determines both performance and cost.

Push-down (runs in BigQuery):
- Filters → WHERE clauses
- Aggregations → GROUP BY + aggregate functions
- LOD expressions → nested subquery JOINs
- COUNTD → exact COUNT(DISTINCT)

Never pushed down (runs in Tableau's memory):
- Table calculations (RUNNING_SUM, RANK, WINDOW_AVG, INDEX, LOOKUP)
- Any calculation using PREVIOUS_VALUE or LOOKUP

## Quick reference: the five-layer optimization stack

1. **Materialized views** — pre-aggregate and flatten; smart tuning rewrites Tableau's queries automatically
2. **BI Engine** — caches working set in memory for sub-second response on eligible queries
3. **Partition enforcement** — `require_partition_filter = true` on table + data source filter in Tableau
4. **Query governors** — BigQuery custom daily quotas + Tableau Server query time limits
5. **INFORMATION_SCHEMA monitoring** — close the feedback loop on cost, slots, and acceleration

---

## Reference files

Load the appropriate reference file based on the user's question:

| File | Load when... |
|------|-------------|
| `references/sql-pushdown.md` | Questions about SQL generation, Custom SQL, LODs, COUNTD, context filters, query fusion |
| `references/cost-control.md` | Questions about partitions, data source filters, BI Engine, materialized views, cost estimation, query governors |
| `references/auth-and-admin.md` | Questions about service accounts, OAuth, credential management, Tableau Server config |
| `references/information-schema.md` | Needs monitoring SQL — copy-paste ready queries for partitions, expensive jobs, slot usage, MV health |

---

## Key rules (always apply, no need to load references)

### Never use Custom SQL when a BigQuery View will do
Custom SQL disables Tableau's join culling, query fusion, and relationship-model optimizations.
A BigQuery view is a drop-in replacement that enables all optimizations:
```sql
CREATE OR REPLACE VIEW `project.dataset.v_daily_sales` AS
SELECT DATE(order_timestamp) AS order_date, region, SUM(amount) AS total_sales
FROM `project.dataset.raw_orders`
GROUP BY 1, 2;
```
Connect Tableau natively to `v_daily_sales` — join culling, query fusion, and partition pruning all work.

### LIMIT does not reduce bytes scanned
`SELECT * FROM big_table LIMIT 100` scans the entire table on standard BigQuery tables.
Exception: clustered tables can prune blocks.

### Green pill dates prune partitions; blue pill dates often don't
Continuous (green) date fields generate `>=` / `<=` comparisons → partition pruning works.
Discrete (blue) date fields generate `EXTRACT(YEAR FROM date_col) = 2024` → no pruning.

### Cost formula
```
Cost ($) = (bytes_processed / 1024⁴) × $6.25
```
First 1 TiB/month free per billing account. Minimum charge: 10 MB per table referenced.

| Scan size | Cost (free tier exhausted) |
|-----------|---------------------------|
| 100 GB    | $0.57                     |
| 1 TB      | $5.68                     |
| 10 TB     | $56.84                    |

---

## Workflow: diagnosing a slow or expensive Tableau dashboard

1. **Check BigQuery Query History** — sort by bytes billed, find the expensive queries
2. **Run the top-jobs INFORMATION_SCHEMA query** (see `references/information-schema.md`)
3. **Is the query scanning the full table?** → Partition filter missing — go to `references/cost-control.md`
4. **Is the query re-scanning the same data multiple times?** → LOD expressions without materialized views — go to `references/sql-pushdown.md`
5. **Is BI Engine not accelerating?** → Check project/region match and preferred tables list — go to `references/cost-control.md`
6. **Is it a table calculation causing large data transfers?** → Replace with LOD expression — go to `references/sql-pushdown.md`
