# Data Ingestion & I/O

## CSV
```python
# Large CSV with chunks
it = pd.read_csv("big.csv", chunksize=250_000)
df = pd.concat(it, ignore_index=True)
```
- Specify dtypes explicitly — inference guesses wrong (IDs become int, losing leading zeros)
- Watch for encoding/locale issues (decimal commas, separators)
- Preview first: sample rows, schema summary before loading full file

## JSON
```python
df = pd.read_json("events.jsonl", lines=True)  # JSON Lines
```
- Nested JSON needs normalization — plan it early, validate expected keys
- "records" vs "table" vs "lines" orientation confusion is common
- Enforce explicit datetime parsing

## Parquet
```python
df = pd.read_parquet("events.parquet", engine="pyarrow",
    filters=[("event_date", ">=", "2026-01-01")])
```
- Columnar, compressed, typed — ideal for analytics
- Filter pushdown behavior is engine-dependent (pyarrow vs fastparquet)
- Enforce dtypes before writing (especially categorical/string)

## SQL
```python
from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg://user:pwd@host:5432/db")
query = text("SELECT * FROM sales WHERE date >= :start AND date < :end")

with engine.begin() as conn:
    df = pd.read_sql(query, conn, params={"start": "2026-01-01", "end": "2026-02-01"})
```
- ALWAYS parameterize queries — never concatenate strings
- Use context managers for connection lifecycle
- Push filters/aggregations to the database, not Python

## APIs (HTTP)
```python
import httpx, asyncio

async def fetch(url):
    async with httpx.AsyncClient(timeout=10.0) as client:
        r = await client.get(url)
        r.raise_for_status()
        return r.json()

results = asyncio.run(asyncio.gather(*(fetch(u) for u in urls)))
```
- Always set timeouts
- Handle retries with backoff
- Prefer allowlisted URLs over arbitrary endpoints
