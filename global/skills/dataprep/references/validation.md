# Validation — 4-Tier Parity Framework

Migrations succeed when the new pipeline produces byte-identical output to the legacy one for the same input. This framework gates a migration through four checks, each cheap to add and decisive about whether to proceed.

Run all four against a representative production sample (or full historical data, if feasible). Don't cut over without all four passing.

## Tier 1 — Schema parity

**What:** compare the column list and types of the two outputs.

**Why:** if columns are missing, renamed, or typed differently, downstream consumers will fail regardless of value-level correctness.

```python
# pandas
def compare_schema(legacy_df, new_df):
    legacy = list(zip(legacy_df.columns, legacy_df.dtypes.astype(str)))
    new = list(zip(new_df.columns, new_df.dtypes.astype(str)))
    if legacy != new:
        for l, n in zip(legacy, new):
            if l != n:
                print(f"MISMATCH: legacy={l}  new={n}")
        return False
    return True

# PySpark
legacy_df.schema == new_df.schema  # exact equality, including nullability

# BigQuery
SELECT column_name, data_type FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = '<table>'
ORDER BY ordinal_position;
# Run for both tables, diff
```

**If it fails:** stop. Schema must match before any value check is meaningful.

## Tier 2 — Row count match

**What:** total row count of both outputs.

**Why:** catches filter, join, dedup, and null-propagation differences. If row counts diverge, something semantic is off — usually a join cardinality issue or a null-dropping aggregation.

```python
# pandas
assert len(legacy_df) == len(new_df), f"Row count mismatch: {len(legacy_df)} vs {len(new_df)}"

# PySpark
legacy_df.count() == new_df.count()

# BigQuery
SELECT (SELECT COUNT(*) FROM legacy_table) AS legacy_count,
       (SELECT COUNT(*) FROM new_table) AS new_count;
```

**If it fails:** investigate the join keys, filter conditions, and null handling. Often a `LEFT JOIN` accidentally became `INNER JOIN` or vice versa.

## Tier 3 — Row-level MD5 checksum match

**What:** concatenate every field in each row, hash with MD5, compare the sorted list of hashes between legacy and new.

**Why:** if schemas match and counts match, this is the deepest pass-or-fail check. Two datasets with the same checksums are byte-identical.

```python
# pandas
import hashlib
def row_hash(row):
    return hashlib.md5('|'.join(str(v) for v in row).encode()).hexdigest()

legacy_hashes = sorted(legacy_df.apply(row_hash, axis=1).tolist())
new_hashes = sorted(new_df.apply(row_hash, axis=1).tolist())
assert legacy_hashes == new_hashes

# PySpark
from pyspark.sql.functions import md5, concat_ws
hashed = lambda df: df.withColumn('_hash', md5(concat_ws('|', *df.columns)))
legacy_h = hashed(legacy_df).select('_hash').rdd.flatMap(lambda x: x).collect()
new_h = hashed(new_df).select('_hash').rdd.flatMap(lambda x: x).collect()
assert sorted(legacy_h) == sorted(new_h)

# BigQuery
WITH legacy_hash AS (
  SELECT TO_HEX(MD5(CONCAT_WS('|', CAST(col1 AS STRING), CAST(col2 AS STRING), ...))) AS h
  FROM legacy_table
),
new_hash AS (
  SELECT TO_HEX(MD5(CONCAT_WS('|', CAST(col1 AS STRING), CAST(col2 AS STRING), ...))) AS h
  FROM new_table
)
SELECT
  (SELECT COUNT(*) FROM legacy_hash) AS legacy_count,
  (SELECT COUNT(*) FROM new_hash) AS new_count,
  (SELECT COUNT(*) FROM legacy_hash l JOIN new_hash n ON l.h = n.h) AS matching;
```

**Watch out for:** floating-point representation differences (`1.0` vs `1`), trailing zeros in decimals, timezone normalization, null vs empty string. Normalize before hashing.

**If it fails:** proceed to Tier 4 to find what specifically diverged.

## Tier 4 — Cell-level diff scan

**What:** for the rows where checksums diverged, scan cell-by-cell and report (row_id, column_name, legacy_value, new_value) for each discrepancy.

**Why:** isolates the exact transformation bug. Usually one of: rounding, null handling, timezone, type coercion.

```python
# pandas (assumes a stable row identifier — pick one or create row_number)
merged = legacy_df.merge(new_df, on='row_id', suffixes=('_legacy', '_new'))
for col in legacy_df.columns:
    if col == 'row_id': continue
    diff_mask = merged[f'{col}_legacy'] != merged[f'{col}_new']
    if diff_mask.any():
        sample = merged.loc[diff_mask, ['row_id', f'{col}_legacy', f'{col}_new']].head(20)
        print(f"\nDiscrepancies in {col}:")
        print(sample)

# BigQuery
SELECT
  COALESCE(l.row_id, n.row_id) AS row_id,
  'col_name' AS column,
  l.col_name AS legacy_value,
  n.col_name AS new_value
FROM legacy_table l
FULL OUTER JOIN new_table n USING (row_id)
WHERE NOT (l.col_name IS NULL AND n.col_name IS NULL)
  AND (l.col_name != n.col_name OR l.col_name IS NULL OR n.col_name IS NULL)
LIMIT 100;
```

**Most common findings, in order:**
1. Null vs empty string ('' vs NULL) — fix with coalesce/fillna
2. Decimal precision differences (rounding) — fix with explicit precision
3. Timezone shifts on dates — fix with timezone-naive types
4. Whitespace differences (trimmed vs not) — fix with explicit trim
5. Case differences — fix with explicit upper/lower

## Putting it together

Wrap all four tiers in a single function:

```python
def validate_migration(legacy_df, new_df, row_id_col='row_id'):
    if not compare_schema(legacy_df, new_df):
        return 'TIER 1 FAIL — schema mismatch'
    if len(legacy_df) != len(new_df):
        return f'TIER 2 FAIL — counts {len(legacy_df)} vs {len(new_df)}'
    if not checksums_match(legacy_df, new_df):
        return 'TIER 3 FAIL — see cell-level diff'
    return 'PARITY VERIFIED'
```

Run this in CI. The output is your cutover green light.
