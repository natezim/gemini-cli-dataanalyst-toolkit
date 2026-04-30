# bq CLI & gcloud storage Reference

## bq query — critical flags

### --format is a GLOBAL flag — must come BEFORE the command
```bash
# CORRECT
bq --format=json query --nouse_legacy_sql 'SELECT 1'

# WRONG — silently ignored
bq query --format=json --nouse_legacy_sql 'SELECT 1'
```

### Default SQL is legacy — always use --nouse_legacy_sql
Set permanently in ~/.bigqueryrc:
```ini
[query]
--use_legacy_sql=false
--max_rows=10000
--maximum_bytes_billed=50000000000

[mk]
--use_legacy_sql=false
```

### --dry_run: free validation and cost estimation — MANDATORY before expensive queries
```bash
BYTES=$(bq query --nouse_legacy_sql --dry_run \
  'SELECT * FROM `project.dataset.table`' 2>&1 | \
  grep -oP '\d+(?= bytes)')
echo "Cost estimate: \$$(echo "scale=4; $BYTES/1099511627776*6.25" | bc)"
```
If estimated scan >1 TB ($6.25+), STOP and warn the user before executing.

### --maximum_bytes_billed: ALWAYS set this as a cost guardrail
Fails immediately with zero cost if query would exceed limit. Set globally in .bigqueryrc.
```bash
bq query --nouse_legacy_sql --maximum_bytes_billed=10000000000 'SELECT ...'
```

### --parameter: safe parameterized queries
```bash
# Named
bq query --nouse_legacy_sql \
  --parameter='min_date:DATE:2026-01-01' \
  --parameter='region::us-east' \
  'SELECT * FROM `dataset.events` WHERE event_date >= @min_date AND region = @region'

# Array
bq query --nouse_legacy_sql \
  --parameter='states:ARRAY<STRING>:["WA","OR","CA"]' \
  'SELECT * FROM `dataset.users` WHERE state IN UNNEST(@states)'
```
Parameters require --nouse_legacy_sql. They prevent SQL injection and enable plan caching.

### --nosync: async submit (returns job ID, not results)
Exit code 0 means SUBMITTED, not succeeded. Must poll with bq show -j.
```bash
JOB_ID=$(bq --nosync query --nouse_legacy_sql 'SELECT 1' 2>&1 \
  | awk '/Successfully started/{print $NF}')
```

### --max_rows defaults to 100 — silently truncates output
All rows are still scanned and billed. Set higher for data exports:
```bash
bq query --nouse_legacy_sql --max_rows=10000 'SELECT ...'
```

### --label for cost attribution
```bash
bq query --nouse_legacy_sql \
  --label=pipeline:daily_etl --label=team:data_eng \
  'SELECT ...'
```

### --job_id for idempotent retries
Prevents duplicate inserts if ETL crashes and retries.
```bash
bq query --nouse_legacy_sql --job_id=etl_orders_2026_04_03 'INSERT INTO ...'
```

### Run queries from files — use stdin redirection
```bash
# CORRECT — handles comments and special characters
bq query --nouse_legacy_sql < etl_pipeline.sql

# WRONG — breaks on -- comments (interpreted as CLI flags)
bq query --nouse_legacy_sql "$(cat etl_pipeline.sql)"
```

### stderr vs stdout
stdout = query results. stderr = status messages + errors.
```bash
bq --format=json query --nouse_legacy_sql 'SELECT 1' 2>/dev/null | jq '.[].n'
```

## bq head — FREE data preview (no query job, no cost)

```bash
bq head --max_rows=10 --selected_fields=user_id,event_type dataset.events
bq head --max_rows=5 'dataset.events$20260401'  # partition-specific
```
A SELECT * LIMIT 10 on a 1 TiB table costs ~$6.25. bq head costs $0.

## Job management

### bq show -j — key JSON paths
```bash
bq show -j --format=json "$JOB_ID" | jq -r '.statistics.query.totalBytesBilled'
bq show -j --format=json "$JOB_ID" | jq '.status.errorResult'
```
Key paths: `.status.state`, `.status.errorResult`, `.statistics.totalBytesProcessed`,
`.statistics.query.totalBytesBilled`, `.statistics.totalSlotMs`, `.configuration.query.query`

### bq wait: timeout ≠ failure (same exit code)
After timeout, check `bq show -j` to determine actual state. Job continues running.

### bq cancel: best-effort, still billed for bytes processed before cancellation

## Table operations

### bq mk — partitioned + clustered table
```bash
bq mk --table \
  --schema='ts:TIMESTAMP,user_id:STRING,event:STRING' \
  --time_partitioning_field=ts --time_partitioning_type=DAY \
  --time_partitioning_expiration=7776000 \
  --clustering_fields=user_id,event \
  --require_partition_filter=true \
  dataset.events
```

### bq rm — single-quote partition decorators (bash $ expansion)
```bash
bq rm -f -t 'dataset.events$20260101'
```

### bq cp — cross-project, single partition, time travel
```bash
bq cp -f project_a:dataset.table project_b:dataset.table
bq cp -f 'dataset.table$20260101' 'other.table$20260101'
TIMESTAMP_MS=$(date -d '1 hour ago' +%s000)
bq cp "dataset.table@${TIMESTAMP_MS}" dataset.table_restored
```

### bq show --schema — export reusable schema
```bash
bq show --schema --format=prettyjson dataset.table > schema.json
bq mk --table --schema=schema.json other_dataset.table_copy
```

### bq ls defaults to 50 results — always set --max_results
```bash
bq ls --format=json --max_results=10000 project:dataset
```

### bq update --expiration is SECONDS FROM NOW, not epoch
```bash
bq update --expiration=604800 dataset.temp_table  # 7 days
bq update --expiration=0 dataset.table            # remove expiration
```

## Data loading

### bq load
```bash
bq load --source_format=PARQUET \
  --time_partitioning_type=DAY --time_partitioning_field=event_date \
  --clustering_fields=user_id,event_type \
  --schema_update_option=ALLOW_FIELD_ADDITION --replace \
  project:dataset.events 'gs://bucket/data/*.parquet'
```
Gotchas: --autodetect misidentifies postal codes as INT64. --replace removes CMEK encryption.
DATETIME columns break Parquet round-trips (use Avro with --use_avro_logical_types).

### EXPORT DATA — better than extract + destination_table
```bash
bq query --nouse_legacy_sql '
EXPORT DATA OPTIONS(
  uri="gs://bucket/results/data-*.parquet",
  format="PARQUET", compression="ZSTD", overwrite=true
) AS SELECT * FROM `project.analytics.events` WHERE DATE(created_at) = "2026-04-01"'
```

### Extract format/compression matrix
| Compression | CSV | JSON | Avro | Parquet |
|---|---|---|---|---|
| GZIP | Yes | Yes | No | Yes |
| SNAPPY | No | No | Yes | Yes |
| ZSTD | No | No | No | Yes |

Wildcard `*` mandatory for tables >1 GB.

## gcloud storage (replaces gsutil)

2-3x faster than gsutil. Parallel by default.

```bash
gcloud storage cp -r ./data/ gs://bucket/data/          # upload
gcloud storage cp -n gs://bucket/** ./local/             # download, skip existing
gcloud storage cat -r 0-999 gs://bucket/huge-log.txt    # byte-range read
gcloud storage rsync ./local gs://bucket/ -r --dry-run   # ALWAYS dry-run first
gcloud storage du -s --readable-sizes gs://bucket
```

Signed URLs without key files:
```bash
gcloud storage sign-url gs://bucket/report.pdf \
  --impersonate-service-account=sa@project.iam.gserviceaccount.com \
  --duration=2h
```

## Authentication

### ADC search order: GOOGLE_APPLICATION_CREDENTIALS → well-known file → metadata server
Stale GOOGLE_APPLICATION_CREDENTIALS overrides everything including login.

### gcloud auth login vs gcloud auth application-default login
Two separate credential stores. Need BOTH for local dev.

### bq impersonation via gcloud config (no bq flag exists)
```bash
gcloud config set auth/impersonate_service_account sa@project.iam.gserviceaccount.com
bq query --nouse_legacy_sql 'SELECT 1'  # runs as service account
```
Recommended over service account keys since May 2024.

### Named configurations prevent wrong-project deploys
```bash
gcloud config configurations create prod
gcloud config set project my-prod-project
gcloud config configurations activate prod  # affects bq too
```

## Production .bigqueryrc template
```ini
--location=US
--headless=true

[query]
--use_legacy_sql=false
--max_rows=10000
--maximum_bytes_billed=50000000000

[mk]
--use_legacy_sql=false
```
`--headless=true` prevents interactive prompts that hang CI/CD.

## Monitoring queries

### Most expensive recent queries
```sql
SELECT job_id, user_email,
  ROUND(total_bytes_billed / 1099511627776 * 6.25, 4) AS cost_usd,
  CONCAT(SUBSTR(query, 1, 100), "...") AS query_preview
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = "QUERY" AND statement_type != "SCRIPT"
  AND state = "DONE" AND total_bytes_billed > 0
ORDER BY total_bytes_billed DESC LIMIT 10
```
Always exclude `statement_type = 'SCRIPT'` to avoid double-counting.

### Per-partition sizes
```sql
SELECT partition_id, total_rows, total_logical_bytes, storage_tier
FROM `dataset.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = "events" ORDER BY partition_id DESC
```

### __TABLES__ metatable (faster than INFORMATION_SCHEMA for quick audits)
```sql
SELECT table_id,
  CASE type WHEN 1 THEN "TABLE" WHEN 2 THEN "VIEW" WHEN 3 THEN "EXTERNAL" END AS type,
  row_count, ROUND(size_bytes/POW(1024,3),3) AS size_gib,
  TIMESTAMP_MILLIS(last_modified_time) AS last_modified
FROM `project.dataset.__TABLES__` ORDER BY size_bytes DESC
```

### DDL recreation (reverse-engineer any table)
```sql
SELECT table_name, ddl FROM `dataset.INFORMATION_SCHEMA.TABLES`
```

### Single-partition metadata (free, no query)
```bash
bq show --format=prettyjson 'dataset.table$20260101'
```
Returns numBytes, numRows, numLongTermBytes for that partition.

## Scheduled queries

Without `--service_account_name`, queries break when the creating user leaves.
```bash
bq mk --transfer_config \
  --data_source=scheduled_query \
  --target_dataset=analytics \
  --display_name="Daily Revenue" \
  --schedule="every day 06:00" \
  --location=US \
  --service_account_name=scheduler@project.iam.gserviceaccount.com \
  --params='{"query": "SELECT ...", "destination_table_name_template": "daily_{run_date|\"%Y%m%d\"}", "write_disposition": "WRITE_TRUNCATE"}'

# List, trigger, delete
bq ls --transfer_config --transfer_location=US --filter=dataSourceIds:scheduled_query
bq mk --transfer_run --run_time="2026-04-03T06:00:00Z" projects/p/locations/us/transferConfigs/ID
bq rm -f --transfer_config projects/p/locations/us/transferConfigs/ID
```

## Cross-region data movement

Ranked by preference:
1. **Dataset replication** (best): `ALTER SCHEMA dataset ADD REPLICA 'us-east4';`
2. **Data Transfer Service**: `bq mk --transfer_config --data_source=cross_region_copy`
3. **Extract + GCS copy + Load** (manual fallback)

## Workload Identity Federation for GitHub Actions
```yaml
permissions:
  contents: read
  id-token: write  # REQUIRED — silently fails without it
jobs:
  pipeline:
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123/locations/global/workloadIdentityPools/pool/providers/github'
          service_account: 'sa@project.iam.gserviceaccount.com'
      - uses: google-github-actions/setup-gcloud@v2
      - run: bq query --nouse_legacy_sql --project_id=project 'SELECT 1'
```

## Additional flags

### --destination_table
Syntax: `PROJECT:DATASET.TABLE` (colon between project and dataset).
Without `--replace` or `--append_table`, fails if table exists.

### --batch: server-side scheduling priority
Batch queries may be queued up to 24 hours. Don't count against interactive concurrency limit (~100/project). Same cost.

### --format values: pretty, sparse, json, prettyjson, csv, none
`none` is useful for DML (suppresses output).

### --apilog: REST API debugging
```bash
bq --apilog=/tmp/bq-debug.log query --nouse_legacy_sql 'SELECT 1'
```

### bq ls -j: no server-side status filter
Filter client-side with jq:
```bash
bq ls -j -a --format=json -n 100 | jq '[.[] | select(.status.state == "RUNNING")]'
```

### .status.errorResult vs .status.errors[]
`.errorResult` present = job failed. `.errors[]` = non-fatal warnings (job may have succeeded).
