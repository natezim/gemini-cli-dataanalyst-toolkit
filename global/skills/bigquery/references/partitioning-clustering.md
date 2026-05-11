# Partitioning & Clustering

## Partition types

**Date/timestamp partitioning** — most common. Supports DAY, HOUR, MONTH, YEAR granularity.
- Hourly: only TIMESTAMP and DATETIME (not DATE). 4,000 partition limit = ~166 days.
- All partition boundaries are UTC. Non-UTC users scan extra partitions at day boundaries.
- For data >6 months, use daily partitioning + clustering instead of hourly.

**Integer range partitioning** — GENERATE_ARRAY(start, end, interval).
- `start` is inclusive, `end` is exclusive. Add +1 if max value equals end.
- Values at/above `end` land in `__UNPARTITIONED__` — no pruning benefit.
- NULLs go to `__NULL__` partition.
- `(end - start) / interval` must stay under 4,000.
- Profile distribution first: `SELECT APPROX_QUANTILES(col, 100) FROM table`

**STRING column workaround** — partition via synthetic integer:
```sql
ABS(MOD(FARM_FINGERPRINT(country_code), 4000))
```
Must include the same hash expression in WHERE clauses for pruning.

## What breaks partition pruning

These KILL pruning (full table scan):
- Any function on partition column: `WHERE customer_id + 1 BETWEEN 30 AND 50`
- CAST or arithmetic on partition column
- Type mismatch: DATE partition with TIMESTAMP literal
- Subquery predicates: `WHERE event_date = (SELECT MAX(event_date) FROM ...)`
- EXTRACT: `WHERE EXTRACT(YEAR FROM event_date) = 2024`
- MERGE with WHEN NOT MATCHED BY SOURCE

These WORK:
- IN lists with constant values
- OR conditions with constants
- DECLARE/SET variable, then use in WHERE (pre-compute subqueries)

**Verify pruning**: check `statistics.query.totalPartitionsProcessed` in execution plan.
Compare bytes_processed to expected partition size.

## Clustering

Acts like a composite index — benefits only on **left prefix**.
Clustering on `(A, B, C)`: filtering on just B and C gives **zero** block pruning.

Rules:
- Put most frequently filtered column first.
- STRING columns clustered on first **1,024 characters** only.
- Tables under **1 GB** see negligible clustering benefit.
- For long strings (URLs, JSON), extract shorter keys.
- Can cluster on STRUCT fields: `user.user_id`. Cannot cluster inside ARRAYs.

**ALTER TABLE to add clustering does NOT recluster existing data.**
Only new writes benefit. To recluster existing data:
```sql
CREATE OR REPLACE TABLE dataset.table
PARTITION BY date_col
CLUSTER BY col_a, col_b
AS SELECT * FROM dataset.table
```

Auto-reclustering is free and autonomous (no slot consumption).

**Audit clustering**:
```sql
SELECT table_name, column_name, clustering_ordinal_position
FROM INFORMATION_SCHEMA.COLUMNS
WHERE clustering_ordinal_position IS NOT NULL
ORDER BY table_name, clustering_ordinal_position
```

Compare dry-run estimated bytes vs actual bytes processed to measure effectiveness.
