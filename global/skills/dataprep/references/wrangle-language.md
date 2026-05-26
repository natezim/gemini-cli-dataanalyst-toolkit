# Wrangle DSL — Operator Registry

Wrangle is declarative — each statement is one logical transform. The Dataprep engine compiles the sequence into Beam (Dataflow) or SQL (BigQuery) at execution time.

## Statement structure

```
<verb> <param>: <value>  <param>: <value>  ...
```

Verbs operate on columns (`col:`), rows (`row:`), or the whole dataset. Common parameters: `col`, `value`, `as` (alias), `type`, `row`, `condition`.

## Operator → target translation registry

### Type conversion

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `settype col: X type: 'Integer'` | `df['X'] = df['X'].astype('Int64')` | `df.withColumn('X', col('X').cast('integer'))` | `CAST(X AS INT64)` |
| `settype col: X type: 'Decimal'` | `df['X'] = df['X'].astype(float)` | `df.withColumn('X', col('X').cast(DecimalType(38, 9)))` | `CAST(X AS NUMERIC)` |
| `settype col: X type: 'Datetime'` | `df['X'] = pd.to_datetime(df['X'])` | `df.withColumn('X', col('X').cast(TimestampNTZType()))` | `CAST(X AS DATETIME)` |
| `settype col: X type: 'String'` | `df['X'] = df['X'].astype(str)` | `df.withColumn('X', col('X').cast('string'))` | `CAST(X AS STRING)` |

**Use `Int64` (capital I) in pandas** for nullable integers — `int` doesn't support NaN.

### Derive (compute new column)

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `derive value: upper(c) as: 'n'` | `df['n'] = df['c'].str.upper()` | `df.withColumn('n', upper(col('c')))` | `UPPER(c) AS n` |
| `derive value: lower(c) as: 'n'` | `df['n'] = df['c'].str.lower()` | `df.withColumn('n', lower(col('c')))` | `LOWER(c) AS n` |
| `derive value: length(c) as: 'n'` | `df['n'] = df['c'].str.len()` | `df.withColumn('n', length(col('c')))` | `LENGTH(c) AS n` |
| `derive value: trim(c) as: 'n'` | `df['n'] = df['c'].str.strip()` | `df.withColumn('n', trim(col('c')))` | `TRIM(c) AS n` |
| `derive value: substring(c, 0, 3) as: 'n'` | `df['n'] = df['c'].str[:3]` | `df.withColumn('n', substring(col('c'), 1, 3))` | `SUBSTR(c, 1, 3) AS n` |
| `derive value: a + b as: 'n'` | `df['n'] = df['a'] + df['b']` | `df.withColumn('n', col('a') + col('b'))` | `a + b AS n` |
| `derive value: dateformat(t, 'YYYY-MM-DD') as: 'd'` | `df['d'] = pd.to_datetime(df['t']).dt.strftime('%Y-%m-%d')` | `df.withColumn('d', date_format(col('t'), 'yyyy-MM-dd'))` | `FORMAT_DATE('%Y-%m-%d', DATE(t)) AS d` |

PySpark `substring` is 1-indexed; pandas `.str[:N]` is 0-indexed. Easy off-by-one.

### Conditional

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `derive value: if(c == 0, null, v) as: 'n'` | `df['n'] = np.where(df['c']==0, np.nan, df['v'])` | `df.withColumn('n', when(col('c')==0, lit(None)).otherwise(col('v')))` | `CASE WHEN c = 0 THEN NULL ELSE v END AS n` |
| `derive value: case(c == 'a', 1, c == 'b', 2, 0) as: 'n'` | `df['n'] = np.select([df['c']=='a', df['c']=='b'], [1, 2], default=0)` | `df.withColumn('n', when(col('c')=='a', 1).when(col('c')=='b', 2).otherwise(0))` | `CASE c WHEN 'a' THEN 1 WHEN 'b' THEN 2 ELSE 0 END AS n` |

### Drop / keep columns

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `drop col: X` | `df = df.drop(columns=['X'])` | `df = df.drop('X')` | `SELECT * EXCEPT(X)` |
| `drop col: X,Y,Z` | `df = df.drop(columns=['X', 'Y', 'Z'])` | `df = df.drop('X', 'Y', 'Z')` | `SELECT * EXCEPT(X, Y, Z)` |
| `keep col: A,B,C` | `df = df[['A', 'B', 'C']]` | `df = df.select('A', 'B', 'C')` | `SELECT A, B, C` |
| `rename col: old as: 'new'` | `df = df.rename(columns={'old': 'new'})` | `df = df.withColumnRenamed('old', 'new')` | `SELECT old AS new, ...` |

### Filter rows

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `filter row: c > 100` | `df = df[df['c'] > 100]` | `df = df.filter(col('c') > 100)` | `WHERE c > 100` |
| `filter row: ismissing([c])` | `df = df[df['c'].isna()]` | `df = df.filter(col('c').isNull())` | `WHERE c IS NULL` |
| `keep row: c > 100` | (same as filter — keeps matching) | (same) | (same) |
| `delete row: c > 100` | `df = df[~(df['c'] > 100)]` | `df = df.filter(~(col('c') > 100))` | `WHERE NOT (c > 100)` |

### Join

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `join right: B keys: (L, R) type: inner` | `pd.merge(A, B, left_on='L', right_on='R', how='inner')` | `A.join(B, A.L == B.R, 'inner')` | `... INNER JOIN B ON A.L = B.R` |
| `join right: B keys: (L, R) type: left` | `pd.merge(A, B, left_on='L', right_on='R', how='left')` | `A.join(B, A.L == B.R, 'left')` | `... LEFT JOIN B ON A.L = B.R` |
| `join right: B keys: (L, R) type: right` | `pd.merge(A, B, left_on='L', right_on='R', how='right')` | `A.join(B, A.L == B.R, 'right')` | `... RIGHT JOIN B ON A.L = B.R` |
| `join right: B keys: (L, R) type: full` | `pd.merge(A, B, left_on='L', right_on='R', how='outer')` | `A.join(B, A.L == B.R, 'full')` | `... FULL OUTER JOIN B ON A.L = B.R` |

### Aggregate / pivot

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `aggregate value: sum(c) group: g` | `df.groupby('g')['c'].sum().reset_index()` | `df.groupBy('g').agg(sum('c'))` | `SELECT g, SUM(c) FROM ... GROUP BY g` |
| `aggregate value: count(*) group: g` | `df.groupby('g').size().reset_index(name='count')` | `df.groupBy('g').count()` | `SELECT g, COUNT(*) FROM ... GROUP BY g` |
| `aggregate value: avg(c), max(c) group: g` | `df.groupby('g').agg(avg=('c','mean'), max=('c','max'))` | `df.groupBy('g').agg(avg('c'), max('c'))` | `SELECT g, AVG(c), MAX(c) FROM ... GROUP BY g` |
| `pivot col: c group: g value: v` | `df.pivot_table(index='g', columns='c', values='v')` | `df.groupBy('g').pivot('c').agg(first('v'))` | (no direct equivalent — use `CASE WHEN` per pivoted value) |

### Deduplicate

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `deduplicate` | `df = df.drop_duplicates()` | `df = df.dropDuplicates()` | `SELECT DISTINCT * FROM ...` |
| `deduplicate col: A,B` | `df = df.drop_duplicates(subset=['A','B'])` | `df = df.dropDuplicates(['A','B'])` | (use `ROW_NUMBER() OVER (PARTITION BY A, B)` and filter) |

### Replace / clean

| Wrangle | pandas | PySpark | BigQuery SQL |
|---|---|---|---|
| `replace col: c on: 'old' with: 'new'` | `df['c'] = df['c'].str.replace('old', 'new', regex=False)` | `df.withColumn('c', regexp_replace(col('c'), 'old', 'new'))` | `REPLACE(c, 'old', 'new')` |
| `replace col: c on: /\\d+/ with: 'X'` | `df['c'] = df['c'].str.replace(r'\\d+', 'X', regex=True)` | `df.withColumn('c', regexp_replace(col('c'), '\\d+', 'X'))` | `REGEXP_REPLACE(c, r'\\d+', 'X')` |
| `set col: c value: 0 row: ismissing([c])` | `df['c'] = df['c'].fillna(0)` | `df.fillna({'c': 0})` | `COALESCE(c, 0)` |

### Untranslatable / partial

These don't have clean equivalents — flag them and propose manual handling:

- `NUMFORMAT` with locale-specific patterns — write a custom Python function
- `transformByExample` (TBE) steps — these store mapping files in `*.data` artifacts; export the mapping and rebuild as a dictionary lookup
- Custom user-defined cluster-clean values — same; the mapping is in the `.data` files
- Wrangle's loose null/empty-string equivalence — explicit `coalesce` / `fillna` / `IFNULL` everywhere
