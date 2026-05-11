# Performance & Scaling

## Vectorization (default optimization)
```python
import numpy as np
X = np.random.randn(10000, 50)
mu = X.mean(axis=0)
sigma = X.std(axis=0) + 1e-8
Xz = (X - mu) / sigma  # broadcasting: (10000,50) - (50,)
```
- Broadcasting runs loops in C instead of Python — orders of magnitude faster
- NEVER use `.apply(lambda ...)` in pandas when a vectorized operation exists
- Watch for huge temporaries — sometimes chunking is better

## Profiling
```python
import cProfile, pstats
cProfile.run("run()", "profile.out")
p = pstats.Stats("profile.out")
p.sort_stats("cumtime").print_stats(30)
```
Profile with real data sizes — hot spots shift with scale.
If I/O dominates, fix data format (Parquet > CSV) before optimizing code.

## Numba (JIT for numeric loops)
```python
import numba as nb

@nb.njit
def pairwise_l2(X):
    n = X.shape[0]
    out = np.empty((n, n), dtype=np.float64)
    for i in range(n):
        for j in range(n):
            d = 0.0
            for k in range(X.shape[1]):
                diff = X[i, k] - X[j, k]
                d += diff * diff
            out[i, j] = np.sqrt(d)
    return out
```
Use for numeric loops that can't be vectorized. Stay within supported NumPy subset.
Won't help with pandas-heavy code.

## Concurrency

**Threads** (I/O-bound):
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=8) as ex:
    parts = list(ex.map(load_one, paths))
df = pd.concat(parts, ignore_index=True)
```
GIL limits CPU-bound Python to one thread. Use processes for CPU work.

**Async** (many I/O calls):
```python
import asyncio, httpx

async def fetch(url):
    async with httpx.AsyncClient(timeout=10.0) as client:
        return (await client.get(url)).json()

results = asyncio.run(asyncio.gather(*(fetch(u) for u in urls)))
```

**Processes** (CPU-bound):
Use `ProcessPoolExecutor` or `multiprocessing` for CPU-heavy transforms.
Serialization overhead between processes — worth it only for substantial work.

## Scaling beyond single machine

| Tool | Best for | Rule of thumb |
|------|----------|--------------|
| Dask | Scale pandas-like workflows | If data fits in RAM, use pandas instead |
| PySpark | Large ETL, SQL-like analytics | Enterprise scale, cluster environments |
| Ray | AI workloads, batch inference | Streaming execution, GPU scheduling |

```python
# Dask
import dask.dataframe as dd
ddf = dd.read_csv("logs/*.csv")
result = ddf.groupby("user_id").amount.sum().compute()
```
Don't use Dask when pandas would work — Dask's own docs say this.
