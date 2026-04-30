---
name: bigquery
description: >
  Expert BigQuery optimization and best practices. Use this skill whenever the user is
  writing BigQuery SQL, optimizing queries, designing schemas, managing costs, debugging
  execution plans, working with partitions/clustering, using window functions, writing
  procedural SQL, or asking about BigQuery features and pricing. Also use when reviewing
  queries for performance, estimating costs, or troubleshooting slow/expensive queries.
---

# BigQuery Expert

Production-grade BigQuery optimization. Covers partitioning, clustering, joins, aggregation,
window functions, cost control, schema design, scripting, security, and 2024-2026 features.

## Critical rules — always apply

### Never SELECT *
Always specify columns. BigQuery is columnar — each column referenced is a full scan of that
column. Unused columns waste bytes and money.

### Always LIMIT exploratory queries
LIMIT does NOT reduce bytes scanned on standard tables (exception: clustered tables).
But it reduces data transfer and client memory. Always include it on exploration.

### bq CLI defaults to legacy SQL — always use --nouse_legacy_sql
Or set permanently in ~/.bigqueryrc. Legacy SQL lacks parameterized queries, scripting,
STRUCT types, and most modern functions. Backtick table references fail without this flag.

### Always dry-run first on expensive queries
Use `--dry_run` flag or EXPLAIN to estimate bytes before executing.
If estimated scan >1 TB, stop and warn the user before running.
Always set `--maximum_bytes_billed` as a cost guardrail on every query.

### Partition pruning breaks silently
These kill partition pruning — never do them:
- Functions on partition column: `WHERE customer_id + 1 > 50`
- CAST on partition column
- Type mismatch: DATE partition with TIMESTAMP literal
- Subquery predicates: `WHERE date = (SELECT MAX(date) FROM ...)`
- EXTRACT on partition column: `WHERE EXTRACT(YEAR FROM date) = 2024`

These work:
- IN lists with constants
- OR conditions with constants
- DECLARE/SET variable then use in WHERE

### NOT EXISTS beats NOT IN
NOT IN has a NULL trap — if subquery contains any NULL, returns zero rows silently.
Always use NOT EXISTS. It also short-circuits and handles NULLs correctly.

### CTEs are NOT materialized
Non-recursive CTEs are inlined and re-executed each time referenced.
If a CTE is referenced more than once, use a temporary table instead.

### JOIN table ordering
Place the largest table first, then smallest, then remaining in decreasing size.
BigQuery only auto-reorders under specific conditions. Filter before joining.

### Use LIKE over REGEXP_CONTAINS
LIKE is faster. Only use REGEXP_CONTAINS when regex is truly needed.

### Prefer SQL UDFs over JavaScript UDFs
The optimizer can inline SQL UDFs. JavaScript UDFs require a separate runtime.

### Cost formula
```
Cost ($) = (bytes_processed / 1,099,511,627,776) × $6.25
```
First 1 TiB/month free. Minimum 10 MB per table per query. Prices in TiB not TB.

| Scan | Cost |
|------|------|
| 100 GB | $0.57 |
| 1 TB | $5.68 |
| 10 TB | $56.84 |

---

## Reference files

Load the appropriate reference based on the user's question:

| File | Load when... |
|------|-------------|
| `references/partitioning-clustering.md` | Partition strategy, clustering order, pruning edge cases, hourly partitions, integer range, reclustering |
| `references/joins-aggregation.md` | Join optimization, broadcast vs hash, HLL++ sketches, GROUPING SETS, window functions, QUALIFY |
| `references/cost-optimization.md` | Storage pricing, zombie tables, long-term storage, logical vs physical billing, editions, autoscaling |
| `references/schema-design.md` | JSON type, STRUCT/ARRAY denormalization, search indexes, wildcard tables, PK/FK constraints |
| `references/scripting-advanced.md` | DECLARE/SET, control flow, EXECUTE IMMEDIATE, transactions, recursive CTEs, pipe syntax, 2024-2026 features |
| `references/bq-cli.md` | bq CLI flags, gcloud storage, authentication, job management, data loading/export, .bigqueryrc, monitoring queries |
| `references/security.md` | Row-level security, column-level security, policy tags, dynamic data masking, IAM best practices |
