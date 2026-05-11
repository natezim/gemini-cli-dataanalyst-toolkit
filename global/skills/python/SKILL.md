---
name: python
description: >
  Universal Python skill for data analysis, ML, scripting, and automation. Use when the
  user is: writing Python scripts, doing data analysis with pandas/NumPy, building ML
  pipelines with scikit-learn, creating visualizations, working with APIs, reading/writing
  CSV/JSON/Parquet/SQL, optimizing performance, setting up environments, testing with
  pytest, or building any Python project.
---

# Python Expert

Production-grade Python for data analysis, ML, scripting, and engineering.

## Critical rules — always apply

### Vectorize, don't loop
Use NumPy/pandas vectorized operations instead of Python loops or `.apply(lambda ...)`.
Loops in C (via NumPy) are orders of magnitude faster than Python loops.

### Pipelines prevent data leakage
Always use scikit-learn Pipelines to chain preprocessing + modeling.
Fit on train, transform on test — pipelines handle this correctly.
Never fit_transform on full dataset then split.

### Parameterize SQL, never concatenate
```python
query = text("SELECT * FROM sales WHERE date >= :start")
df = pd.read_sql(query, conn, params={"start": "2026-01-01"})
```
String concatenation = SQL injection risk.

### Type hints at boundaries, not everywhere
Type function signatures (inputs/outputs), tool interfaces, and config classes.
Don't over-type exploratory notebook code. Type hints are NOT runtime enforcement —
pair with Pydantic or pandera for real validation.

### eval() and exec() are never safe with untrusted input
Python docs explicitly state overriding `__builtins__` is NOT a security mechanism.
Never unpickle untrusted data either.

### Explicit dtypes on data ingestion
Don't let pandas guess — specify dtypes, especially for IDs (leading zeros get dropped),
dates (parse explicitly), and categoricals (use `astype("category")` for low-cardinality).

### Always close connections
Use context managers (`with engine.begin() as conn:`) for database connections.
Forgetting disposal is a common leak.

### Assert shapes before expensive operations
Broadcasting errors are silent. `assert X.shape == (n, d)` before matrix operations.

---

## Reference files

| File | Load when... |
|------|-------------|
| `references/data-io.md` | CSV, JSON, Parquet, SQL, API ingestion, chunked reading, filter pushdown |
| `references/analysis-ml.md` | pandas patterns, scikit-learn pipelines, statsmodels, AutoML tools, feature engineering |
| `references/performance.md` | Vectorization, Numba, profiling, concurrency (threads/async/processes), scaling (Dask/Spark/Ray) |
| `references/visualization.md` | matplotlib, seaborn, Plotly, Altair, Bokeh — when to use which, minimal idioms |
| `references/engineering.md` | Testing (pytest), packaging, virtual environments, containers, model serving, CI/CD |
