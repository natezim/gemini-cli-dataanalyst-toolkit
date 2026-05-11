# Tableau Performance Optimization

## Workbook Optimizer (2022.1+)
Built-in tool that flags: excessive visible sheets, excessive filters/sheet, excessive LODs,
excessive layout containers, non-fixed sizing, unused data sources, unused fields,
non-materialized calcs, live connections, data blending, "Only Relevant Values" filters,
nested calcs, long calcs, conditional filter logic. Run it before publishing.

## Dashboard thresholds
- ~4 worksheets per dashboard max (each fires its own query)
- <2,000 marks per view optimal. >4,000 = slow.
- Fixed-size dashboards only — automatic sizing prevents caching.
- Same LOD across sheets enables query batching (8 queries → 2–3).

## Calculated field anti-patterns

**COUNTD**: slowest aggregation type. Replace with FIXED LOD:
`{FIXED [OrderID]: MAX(1)}` then SUM = 5–20x faster.

**Data type hierarchy**: Boolean/Integer > Date > String.
String operations (FIND, REPLACE, REGEX) are expensive.

**CASE vs IF**: CASE is faster. ELSEIF (one word) faster than ELSE IF (two words).

**NOW()/TODAY()**: prevent caching, cannot materialize in extracts.
Use `{FIXED : MAX([Date])}` instead.

**LOD expressions**: generate nested SELECT subqueries. Nested LODs compound cost.
High-cardinality dimensions risk cross-joins.

**Table calculations**: run client-side on full result set. Performance degrades with marks.

**Row-level calcs with parameters**: cannot materialize.
Separate non-parameter portion into standalone calc that CAN materialize.

**MIN/MAX** perform better than AVG/ATTR.

## Filter order of operations (cheapest → most expensive)
1. Extract filters — data never enters extract
2. Data source filters — limit what enters workbook
3. Context filters — temp table, recomputed on every change
4. FIXED LODs / Sets / Top N
5. Dimension filters — WHERE clauses
6. INCLUDE/EXCLUDE LODs
7. Measure filters — HAVING clauses
8. Table calc filters — only hide marks, data still loaded

- "Only Relevant Values" = extremely expensive (requery on every filter change)
- Context filters no longer improve query performance in modern Tableau
  (only serve order-of-operations control for FIXED LODs and Top N)
- Filter actions >> quick filters for performance

## Extract optimization
- **Aggregated extracts** ("Aggregate for visible dimensions") dramatically reduce size
- **Hide All Unused Fields** before creating extract
- Fields hidden after creation not removed until next full refresh
- **Materialize Calculations** for complex string/regex calcs
- Cannot materialize: NOW(), TODAY(), LODs, table calcs, RAWSQL, SCRIPT_*, user functions
- Incremental refresh degrades over time — full refresh weekly/monthly
- Extracts almost always outperform live for datasets under billions of rows

## Server and caching
- Cache modes: "low" (default, cache long), specific minutes, "always/0" (refresh each load)
- **View Acceleration** (2023.1+): precomputes selected workbooks, max 12 jobs/day
- **Cache warming**: schedule subscription after extract refresh, before users arrive
  Pattern: Extract 2 AM → Subscription 5 AM → Users get cached views 8 AM

## Data source best practices
- Avoid cross-database joins (pulls all data locally)
- Use Relationships over Joins or Blends (only queries needed tables)
- Custom SQL disables join culling — use native tables or DB views instead
