# Migration Risks — The Three That Bite

Wrangle's visual abstraction hides semantic differences from Python/Spark/SQL runtimes. Three discrepancies cause **silent data corruption** during migration. Address each explicitly in every translation.

## 1. Temporal string drift

**The trap:** Wrangle's UI lets you treat dates as strings — typing operations, substring extraction, regex — without complaint. The data sits as a string under the hood and only gets coerced to a datetime when you do date-aware operations.

**Why it breaks:**
- PySpark throws `AnalysisException` if you do string operations on a real datetime column (or vice versa).
- pandas silently coerces things in ways that don't match Wrangle (e.g., timezone inference).
- BigQuery's `DATETIME` vs `TIMESTAMP` vs `STRING` are strict — wrong type = either error or wrong result.

**Additional gotcha — timezone naïveté:**
Alteryx/Dataprep datetime variables do NOT carry timezone metadata. Migrating to a timezone-aware target (PySpark `TimestampType`, BQ `TIMESTAMP`) causes implicit UTC conversion — dates shift by hours or roll over a day.

**Fix:** explicit cast to a timezone-naive equivalent.

```python
# PySpark — use TimestampNTZType (no time zone)
from pyspark.sql.types import TimestampNTZType
df = df.withColumn('clean_date', col('raw_date').cast(TimestampNTZType()))

# pandas — keep tz-naive
df['clean_date'] = pd.to_datetime(df['raw_date'], errors='coerce')
# verify: df['clean_date'].dt.tz is None

# BigQuery SQL — use DATETIME (no TZ), not TIMESTAMP
SELECT CAST(raw_date AS DATETIME) AS clean_date
```

**Validation:** for any datetime column, after translation, sample 10 rows and verify the date value matches the Dataprep output **before** the cast happens.

## 2. Decimal precision truncation

**The trap:** Alteryx/Dataprep `FixedDecimal` supports up to **50 digits** of precision. PySpark and BigQuery both cap at **38**.

**Why it breaks:**
- Spark: casting a 50-digit decimal to `DecimalType(38, X)` either truncates silently or fails compilation.
- BigQuery: `NUMERIC` is `DECIMAL(38, 9)`, `BIGNUMERIC` is `DECIMAL(76, 38)` (bigger but still finite).

**Fix:** when a column has high precision, either scale down or store as string.

```python
# PySpark — scale to 38 digits explicitly
from pyspark.sql.types import DecimalType
df = df.withColumn('amount', col('amount').cast(DecimalType(38, 9)))

# Or preserve as string when precision matters more than arithmetic
df = df.withColumn('amount_str', col('amount').cast('string'))

# BigQuery — use BIGNUMERIC for up to 76 digits
CAST(amount AS BIGNUMERIC) AS amount
```

**Validation:** check the source column's actual digit distribution before deciding the precision. If real values fit in 18 digits, you don't need to worry about 38.

## 3. Null propagation discrepancies

**The trap:** Wrangle often treats `NULL` and empty string `''` as equivalent. Python/Spark/SQL enforce SQL-92 semantics where `concat(x, NULL) = NULL`.

**Why it breaks:** a row with one null field in the source can cascade — concatenations produce NULL, aggregations skip NULLs, joins drop rows. Output row counts diverge from the Dataprep version.

**Fix:** explicit coalesce / fillna / IFNULL before any concatenation or NULL-sensitive operation.

```python
# PySpark
from pyspark.sql.functions import concat, coalesce, lit
df = df.withColumn(
    'full_name',
    concat(coalesce(col('first'), lit('')), lit(' '), coalesce(col('last'), lit('')))
)

# pandas
df['full_name'] = df['first'].fillna('') + ' ' + df['last'].fillna('')

# BigQuery SQL
SELECT CONCAT(IFNULL(first, ''), ' ', IFNULL(last, '')) AS full_name
```

**Validation:** row count comparison (Tier 2 in the validation framework) catches this immediately. If row counts match but column nullability differs, suspect cascading nulls.

## Quick mental model

When translating any Wrangle recipe, ask:

- Does any column contain dates? → check timezone-naïveté
- Does any column contain decimals? → check precision ceiling
- Does any operation combine columns or aggregate? → check null handling

If yes to any, the translation needs explicit defensive code, not just operator-for-operator mapping.

## Other known sharp edges

- **`NUMFORMAT`** — locale-specific formatting. No clean equivalent. Write a custom function.
- **Transformation by Example (TBE)** — Wrangle infers the transform from examples; the result lives in `.data` files. Export those, build a lookup dict, apply with `.map()` (pandas) or a UDF (Spark).
- **Manual cluster-clean overrides** — same as TBE; lookup table.
- **Wrangle's "auto-detect" types on import** — never reproduce this in production code. Always declare types explicitly.
