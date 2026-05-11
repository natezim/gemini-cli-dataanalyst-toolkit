# Scripting & Advanced Features

## DECLARE and SET

- Variables support all types: ARRAY, STRUCT, JSON, etc.
- Type inferred from DEFAULT value.
- SET multiple variables from single query:
```sql
DECLARE start_date DATE;
DECLARE end_date DATE;
SET (start_date, end_date) = (
  SELECT AS STRUCT MIN(date), MAX(date) FROM events
);
```
- Max variable size: 1 MB; max total: 10 MB.
- Variables scoped to enclosing BEGIN...END block.

## Control flow

- IF/ELSEIF/ELSE, WHILE, LOOP, FOR...IN with labels.
- Max nesting: 50 levels.
- BREAK and LEAVE are synonymous; CONTINUE and ITERATE synonymous.
- FOR...IN iterates query results — loop variable is STRUCT.
- **Loops are serial — avoid row-by-row iteration on large datasets.**

## EXECUTE IMMEDIATE (dynamic SQL)

```sql
EXECUTE IMMEDIATE FORMAT('SELECT * FROM `%s.%s.%s`', project, dataset, table_name);
```

- USING passes values safely (like query parameters).
- For identifiers (table/column names): use FORMAT — USING cannot parameterize identifiers.
- Cannot nest IF, LOOP, WHILE, BEGIN/END inside EXECUTE IMMEDIATE.
- **Validate/allowlist inputs for FORMAT to prevent injection.**

## Exception handling

```sql
BEGIN
  INSERT INTO prod.table SELECT * FROM staging;
EXCEPTION WHEN ERROR THEN
  SELECT @@error.message;
  ROLLBACK;
END;
```

- Available: `@@error.message`, `@@error.statement_text`, `@@error.formatted_stack_trace`
- RAISE re-throws; `RAISE USING MESSAGE` throws custom.
- Variables in BEGIN NOT accessible in EXCEPTION handler.

## Transactions

- Support INSERT/UPDATE/DELETE/MERGE/TRUNCATE.
- **Permanent DDL NOT allowed** inside transactions.
- Limits: 100 tables, 100,000 partition modifications per transaction.
- Unhandled errors auto-rollback.

## Recursive CTEs

```sql
WITH RECURSIVE hierarchy AS (
  SELECT id, parent_id, name, 0 AS depth FROM orgs WHERE parent_id IS NULL
  UNION ALL
  SELECT o.id, o.parent_id, o.name, h.depth + 1
  FROM orgs o JOIN hierarchy h ON o.parent_id = h.id
  WHERE h.depth < 10  -- always include depth guard
)
SELECT * FROM hierarchy
```

- Max recursion depth: 500 iterations (default).
- Each iteration is a separate execution stage.
- Recursive CTEs ARE materialized (unlike non-recursive).

## System variables

Writable: `@@dataset_id`, `@@time_zone`, `@@query_label`.
Read-only: `@@project_id`, `@@script.bytes_processed`, `@@script.slot_ms`, `@@row_count`.

Session mode: top-level DECLARE'd variables and temp tables persist across submissions.
Reference temp tables explicitly: `_SESSION.table_name` to avoid collisions.

## FOR...IN with dynamic SQL

```sql
FOR tbl IN (
  SELECT table_name FROM `project.dataset.INFORMATION_SCHEMA.TABLES`
  WHERE table_type = 'BASE TABLE'
) DO
  EXECUTE IMMEDIATE FORMAT(
    "SELECT '%s' AS tbl, COUNT(*) AS rows FROM `project.dataset.%s`",
    tbl.table_name, tbl.table_name
  );
END FOR;
```

Labeled loops: `outer_loop: LOOP ... BREAK outer_loop; ... END LOOP;`

## Pipe syntax (GA February 2025)

```sql
FROM events
|> WHERE event_date >= '2024-01-01'
|> AGGREGATE COUNT(*) AS event_count GROUP BY user_id
|> WHERE event_count > 10
|> ORDER BY event_count DESC
```

- Queries start with FROM and chain operators with `|>`.
- New operators: EXTEND, AGGREGATE...GROUP BY, SET, DROP, RENAME.
- Same optimizer, same cost — purely syntactic.
- Eliminates deep nesting and repetitive CTEs.

## 2024-2026 features

**Advanced runtime** (default for all projects, late 2025):
- Improved query execution time and slot usage. Includes short-query optimizations.
- Automatic — no configuration needed.

**History-Based Optimizations** (GA October 2024):
- Learns from prior executions. Auto-improves join ordering, pushdown, semijoin reduction.
- Up to 100x improvement observed. No configuration needed.

**Anti-pattern recognition tool** (2025):
- Scans SQL and provides optimization recommendations.
- Reads from INFORMATION_SCHEMA, outputs to BigQuery table.

**Automatic partition/clustering recommendations**:
- BigQuery recommender system suggests optimal partitioning and clustering columns.
- Check via Cloud Console or `gcloud recommender recommendations list`.

**AI functions** (GA):
- `AI.GENERATE` / `AI.GENERATE_TABLE`: text, image, video, audio, document input.
- `AI.EMBED()`: embedding generation.
- `AI.SIMILARITY()`: semantic similarity scores.
- `AI.FORECAST`: time-series forecasting powered by TimesFM.
- Gemini 3.0 Pro/Flash model support.

**Non-incremental materialized views** (GA April 2024):
- `allow_non_incremental_definition = true` for complex JOINs, UNION ALL, analytics.
- `max_staleness` control. Full refreshes only, no smart tuning.

**VECTOR_SEARCH** (GA):
- Similarity search over ARRAY<FLOAT64> embeddings.
- Distance: COSINE, EUCLIDEAN, DOT_PRODUCT.
- IVF index for small batches; TreeAH for large batches.
- Enables RAG, semantic search natively without external vector DB.

**Continuous queries** (GA):
- Event-driven pipelines with Pub/Sub integration.
- INSERT into BigQuery tables or EXPORT DATA to Pub/Sub topics.
- Max 2 days for user accounts. Minimum 100 slots.

**Legacy SQL deprecated June 1, 2026.**
