# INFORMATION_SCHEMA Monitoring Queries

Copy-paste ready SQL for BigQuery monitoring. All queries use Standard SQL.

**Important notes before running:**
- Always filter `creation_time` (it's the partition column on JOBS views — unfiltered = full scan)
- Region prefix is mandatory for project-level views: use `region-us`, `region-eu`, `region-us-central1`, etc.
- INFORMATION_SCHEMA queries are never cached — you're billed each time you run them
- JOBS data retains **180 days**
- Replace `my_project`, `my_dataset`, `my_table` with actual values

---

## 1. Partition sizes and row counts

```sql
SELECT
    table_name,
    partition_id,
    total_rows,
    ROUND(total_logical_bytes / POW(1024, 3), 2) AS size_gb,
    storage_tier,
    last_modified_time
FROM `my_project.my_dataset.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'my_partitioned_table'
    AND partition_id IS NOT NULL
ORDER BY partition_id DESC;
```

---

## 2. Top 10 most expensive queries (last 30 days)

```sql
SELECT
    job_id,
    user_email,
    creation_time,
    ROUND(total_bytes_billed / POW(1024, 4) * 6.25, 4) AS estimated_cost_usd,
    ROUND(total_bytes_billed / POW(1024, 3), 2) AS gb_billed,
    total_slot_ms,
    cache_hit,
    TIMESTAMP_DIFF(end_time, start_time, SECOND) AS duration_sec,
    statement_type,
    SUBSTR(query, 1, 300) AS query_preview
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND job_type = 'QUERY'
    AND state = 'DONE'
    AND statement_type != 'SCRIPT'
ORDER BY total_bytes_billed DESC
LIMIT 10;
```

---

## 3. Daily cost by user (today)

```sql
SELECT
    user_email,
    COUNT(*) AS query_count,
    ROUND(SUM(total_bytes_processed) / POW(1024, 4), 4) AS tib_scanned,
    ROUND(SUM(total_bytes_processed) / POW(1024, 4) * 6.25, 2) AS estimated_cost_usd,
    ROUND(SUM(total_bytes_billed) / POW(1024, 4) * 6.25, 2) AS billed_cost_usd,
    COUNTIF(cache_hit) AS cached_queries
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_TRUNC(CURRENT_TIMESTAMP(), DAY)
    AND job_type = 'QUERY'
    AND state = 'DONE'
GROUP BY user_email
ORDER BY billed_cost_usd DESC;
```

---

## 4. Hourly slot usage trend (last 7 days)

```sql
SELECT
    TIMESTAMP_TRUNC(period_start, HOUR) AS hour,
    ROUND(SUM(period_slot_ms) / (1000 * 3600), 2) AS slot_hours,
    COUNT(DISTINCT job_id) AS job_count
FROM `region-us`.INFORMATION_SCHEMA.JOBS_TIMELINE_BY_PROJECT
WHERE period_start > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY hour
ORDER BY hour DESC;
```

---

## 5. Table schema with partition and clustering info

```sql
SELECT
    column_name,
    ordinal_position,
    data_type,
    is_nullable,
    is_partitioning_column,
    clustering_ordinal_position
FROM `my_project.my_dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'my_table'
ORDER BY ordinal_position;
```

---

## 6. Table metadata: partitioning type and clustering fields

```sql
SELECT
    table_name,
    table_type,
    creation_time,
    DDL
FROM `my_project.my_dataset.INFORMATION_SCHEMA.TABLES`
WHERE table_name = 'my_table';

-- For partition + cluster details:
SELECT
    table_name,
    partition_expiration_days,
    require_partition_filter,
    clustering_fields
FROM `my_project.my_dataset.INFORMATION_SCHEMA.TABLE_OPTIONS`
WHERE table_name = 'my_table';
```

---

## 7. Materialized view health check

```sql
SELECT
    table_name,
    last_refresh_time,
    refresh_watermark,
    last_refresh_status
FROM `my_project.my_dataset.INFORMATION_SCHEMA.MATERIALIZED_VIEWS`;
```

---

## 8. BI Engine acceleration monitoring (last 1 hour)

```sql
SELECT
    bi_engine_statistics.bi_engine_mode AS acceleration_mode,
    COUNT(*) AS query_count,
    ARRAY_AGG(DISTINCT r.reason_code IGNORE NULLS) AS fallback_reasons
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    UNNEST(bi_engine_statistics.bi_engine_reasons) AS r
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
    AND bi_engine_statistics IS NOT NULL
GROUP BY acceleration_mode;
```

BI Engine mode values:
- `FULL` — query fully accelerated (target: >80% of queries)
- `PARTIAL` — partially accelerated (some operations fell back)
- `DISABLED` — not accelerated (check project/region match and preferred tables list)

---

## 9. Queries missing partition filters (last 24 hours)

Identifies expensive queries that scanned full tables — likely missing partition filters:

```sql
SELECT
    job_id,
    user_email,
    creation_time,
    ROUND(total_bytes_billed / POW(1024, 3), 2) AS gb_billed,
    ROUND(total_bytes_billed / POW(1024, 4) * 6.25, 4) AS estimated_cost_usd,
    SUBSTR(query, 1, 500) AS query_preview
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
    AND job_type = 'QUERY'
    AND state = 'DONE'
    AND total_bytes_billed > 10 * POW(1024, 3)   -- > 10 GB billed
    AND cache_hit = FALSE
ORDER BY total_bytes_billed DESC
LIMIT 20;
```

---

## 10. Cache hit rate (last 7 days)

```sql
SELECT
    DATE(creation_time) AS query_date,
    COUNT(*) AS total_queries,
    COUNTIF(cache_hit) AS cached_queries,
    ROUND(COUNTIF(cache_hit) / COUNT(*) * 100, 1) AS cache_hit_pct,
    ROUND(SUM(IF(NOT cache_hit, total_bytes_billed, 0)) / POW(1024, 4) * 6.25, 2) AS billed_cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    AND job_type = 'QUERY'
    AND state = 'DONE'
GROUP BY query_date
ORDER BY query_date DESC;
```

---

## 11. Recent Tableau-sourced queries (if labeled)

If your org configures BigQuery job labels for Tableau (via Tableau Server config):

```sql
SELECT
    job_id,
    user_email,
    creation_time,
    ROUND(total_bytes_billed / POW(1024, 4) * 6.25, 4) AS cost_usd,
    labels
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
    AND EXISTS (
        SELECT 1 FROM UNNEST(labels)
        WHERE key = 'workload' AND value = 'tableau'
    )
ORDER BY total_bytes_billed DESC;
```

To enable Tableau job labels on Tableau Server:
```bash
tsm configuration set -k native_api.bigquery_job_label_keys -v "workload,workbook,user"
tsm pending-changes apply
```

---

## Quick cost estimation

```sql
-- Estimate cost of a specific query before running it:
-- Use BigQuery dry run (Console: More → Query Settings → "Only estimate bytes processed")
-- Or via CLI: bq query --dry_run --nouse_legacy_sql "SELECT ..."

-- Manual calculation:
-- bytes_processed_from_dry_run / 1,099,511,627,776 * 6.25 = cost in USD
```
