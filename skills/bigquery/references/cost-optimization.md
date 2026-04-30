# Cost Optimization

## Cost formula

```
Cost ($) = (bytes_processed / 1,099,511,627,776) × $6.25
```

- Prices in **TiB** (1024^4 bytes), not TB.
- First 1 TiB/month free per billing account.
- Minimum charge: 10 MB per table referenced, 10 MB per query.
- Cached results: free (same query, same data, within 24 hours).
- Error queries: free (if error before scan).
- LIMIT does NOT reduce bytes scanned (exception: clustered tables).

| Scan | Cost |
|------|------|
| 100 GB | $0.57 |
| 1 TB | $5.68 |
| 10 TB | $56.84 |

## Finding zombie tables

```sql
SELECT ts.table_schema, ts.table_name,
  ts.total_logical_bytes / POW(1024,3) AS size_gb,
  ts.last_modified_time
FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE ts
LEFT JOIN (
  SELECT DISTINCT referenced_tables.table_id
  FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
  UNNEST(referenced_tables) AS referenced_tables
  WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
) qt ON ts.table_name = qt.table_id
WHERE qt.table_id IS NULL
ORDER BY size_gb DESC
```

Teams save 10-30% on storage by removing unused tables.

## Long-term storage timer

90-day timer for long-term pricing **resets on**: DML writes, streaming inserts, loads, copies.
Does **NOT reset on**: queries, exports, metadata updates, view creation.

For partitioned tables: each partition tracked independently.
Single UPDATE on historical table resets to active pricing.
**Design pipelines to be append-only** to preserve long-term pricing.

## Logical vs physical storage billing

Physical billing charges compressed data at 2x logical rate.
Compression ratios typically 2:1 to 16:1, making it cheaper.

- Physical billing charges time-travel and fail-safe bytes separately.
- Switch at dataset level; **14-day wait** between switches.
- 20-70% savings common for structured columnar data.
- Under physical billing: reduce time-travel window to 2 days if recovery not critical.

## Editions pricing

| Edition | Per slot-hour | 100 slots/day |
|---------|-------------|--------------|
| Standard | $0.04 | $96 |
| Enterprise | $0.06 | $144 |
| Enterprise Plus | $0.10 | $240 |

- Autoscales in 50-slot increments, 1-minute minimum billing, 60-second cooldown.
- Set baseline to 0 for dev/test (pay only for actual usage).
- 1-year commitment: ~20% discount. 3-year: ~40%.
- On-demand: $6.25/TiB.

## CTE cost trap

Non-recursive CTEs are NOT materialized — inlined and re-executed each reference.
A CTE referenced 3 times = 3x the cost. Use temporary tables for multi-referenced CTEs:
```sql
CREATE TEMP TABLE staging AS SELECT ...;
-- Now reference staging multiple times at no extra scan cost
```

## Storage pricing

| Type | Price |
|------|-------|
| Active (<90 days) | $0.02/GiB/month (logical) |
| Long-term (>=90 days untouched) | $0.01/GiB/month (automatic) |

Compare logical vs physical billing before switching:
```sql
SELECT table_schema,
  ROUND(SUM(active_logical_bytes)/POW(1024,3) * 0.02
    + SUM(long_term_logical_bytes)/POW(1024,3) * 0.01, 2) AS logical_cost,
  ROUND(SUM(active_physical_bytes)/POW(1024,3) * 0.04
    + SUM(long_term_physical_bytes)/POW(1024,3) * 0.02, 2) AS physical_cost
FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE_BY_PROJECT
GROUP BY table_schema ORDER BY logical_cost DESC
```

Switch: `ALTER SCHEMA dataset SET OPTIONS (storage_billing_model = 'PHYSICAL');`
14-day wait between switches.

## BI Engine

In-memory acceleration for frequently accessed data. No Tableau/app-side config needed.
- Requires Enterprise or Enterprise Plus edition.
- Max 250 GB default reservation.
- Accelerates JOINs with up to 4 dimension tables.
- Billing project must match reservation project.

## Custom daily quotas — critical cost control

Set at project and per-user level. Resets midnight Pacific.
```
IAM & Admin → Quotas → "Query usage per day" → Edit
"Query usage per day per user" → Edit
```
New projects (Sep 2025+) default to 200 TiB/day.

## Materialized view limits

- Max 20 MVs per base table.
- Incremental MVs require fact table as leftmost in JOINs.
- DML (UPDATE/MERGE/DELETE) on base tables invalidates incremental updates.
- Must refresh within 3 days (streaming buffer retention).
- Smart tuning auto-rewrites queries to use MVs transparently.

Health check:
```sql
SELECT table_name, last_refresh_time, refresh_watermark, last_refresh_status
FROM `project.dataset.INFORMATION_SCHEMA.MATERIALIZED_VIEWS`
```

## Monitoring queries

**Daily cost per user:**
```sql
SELECT user_email,
  ROUND(SUM(total_bytes_processed) / POW(1024, 4) * 6.25, 2) AS cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_TRUNC(CURRENT_TIMESTAMP(), DAY)
  AND job_type = 'QUERY' AND state = 'DONE'
GROUP BY user_email ORDER BY cost_usd DESC
```

**Hourly slot usage (7-day trend):**
```sql
SELECT TIMESTAMP_TRUNC(period_start, HOUR) AS hour,
  SUM(period_slot_ms) / (1000 * 3600) AS slot_hours
FROM `region-us`.INFORMATION_SCHEMA.JOBS_TIMELINE_BY_PROJECT
WHERE period_start > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY hour ORDER BY hour DESC
```

**INFORMATION_SCHEMA queries are never cached** — you're billed each time.
Always include WHERE filters. JOBS data retains 180 days.

## UNION complexity limit

~89 UNION ALL operations exhaust the 50M query complexity quota.
Monitor: `SELECT query_info.resource_warning FROM INFORMATION_SCHEMA.JOBS WHERE resource_warning IS NOT NULL`
Workarounds:
- Batch into temp tables (45 + 45 + final UNION).
- Use wildcard tables: `FROM dataset.events_* WHERE _TABLE_SUFFIX BETWEEN ...`
