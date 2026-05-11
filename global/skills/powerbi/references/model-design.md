# Semantic Model Design

## Star schema rules
- Dimension tables → one-to-many → fact tables. Always.
- Denormalize snowflake dimensions into single tables.
- Integer surrogate keys. Proper star schema = ~60% smaller, 10x faster.
- Single-direction filtering (dimension → fact) as default.
- ~90% DAX should be measures, not calculated columns.

## Storage modes

**Import**: data loaded into VertiPaq. Best performance. Requires scheduled refresh.
**DirectQuery**: every interaction = live SQL query. Real-time but slower.
**Dual**: table available in both modes. Critical for dimensions in composite models.
**Direct Lake**: reads Delta/Parquet from OneLake. Import performance without refresh.

## Composite models
Mix Import and DirectQuery in one model:
```
Fact tables (large):        DirectQuery
Aggregation tables:         Import (pre-summarized)
Dimension tables:           Dual (CRITICAL — avoids limited relationships)
Calculated tables:          Always Import
```

Without Dual mode on dimensions, Import-to-DirectQuery joins create **limited relationships**
(data transferred and joined in memory on un-indexed data = extremely slow).

## DirectQuery optimization
Every visual interaction = live SQL query. 10 visuals + 3 slicers = 13+ queries per click.
- Enable query reduction settings ("Add Apply button to slicers")
- Create Import aggregation tables for common summaries
- Set dimensions to Dual storage mode
- User-defined aggregations: Import table handles summary queries, DirectQuery for detail

## Incremental refresh
Create two Power Query parameters: `RangeStart` and `RangeEnd` (case-sensitive, DateTime type).
```m
= Table.SelectRows(Source, each [OrderDate] >= RangeStart and [OrderDate] < RangeEnd)
```
- Requires query folding at the filter step
- Without folding, all data loaded then filtered locally (defeats purpose)
- After publishing with incremental refresh, you cannot download the .pbix
- Service creates date-based partitions and refreshes only the incremental window

## Relationship rules
- One active relationship per pair of tables. Use USERELATIONSHIP for inactive.
- Avoid M:M relationships when possible (create bridge tables).
- Bidirectional = ambiguous filter paths. Use only via CROSSFILTER in specific measures.

## Model size optimization
- Remove unused columns (they're stored even if not in visuals)
- Reduce cardinality: round decimals, bin continuous values
- Disable Auto Date/Time (Options → Current File → Data Load)
- Use calculated tables sparingly (always Import, always stored)
