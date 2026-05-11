---
name: tableau-optimizer
description: >
  Tableau workbook optimization, TWBX/TWB parsing, and Power BI migration. Use this skill
  when the user is: analyzing Tableau workbooks for performance issues, parsing TWB/TWBX XML,
  auditing calculated fields or custom SQL, optimizing dashboard load times, working with
  LOD expressions or table calculations, planning Tableau-to-Power BI migration, scoring
  migration complexity, translating Tableau calcs to DAX, or using the Tableau REST/Metadata
  API for governance automation.
---

# Tableau Optimizer

Production-grade Tableau performance optimization, workbook analysis, and Power BI migration.

## Critical rules — always apply

### Performance thresholds
- Limit worksheets per dashboard to ~4 views (each fires its own query)
- Keep marks per view under 1,000–2,000. Over 4,000 = noticeably slow.
- A 10-column × 33-row text table alone = 3,300 marks.
- Use fixed-size dashboards — automatic sizing prevents caching.

### COUNTD is almost always the bottleneck
COUNTD is one of the slowest aggregation types. Replace with:
```
{FIXED [OrderID]: MAX(1)}
```
Then SUM — gives 5–20x better performance.

### Filter cost ranking (cheapest to most expensive)
1. Extract filters (data never enters extract)
2. Data source filters (limit what enters workbook)
3. Context filters (create temp table, expensive to recompute)
4. FIXED LODs / Sets / Top N
5. Dimension filters (WHERE clauses)
6. INCLUDE/EXCLUDE LODs
7. Measure filters (HAVING clauses)
8. Table calc filters (only hide marks, data still loaded)

"Only Relevant Values" on filters is extremely expensive — avoid it.
Filter actions are more performant than quick filters.

### NOW() and TODAY() prevent caching
Use `{FIXED : MAX([Date])}` instead (data-driven, not time-driven).

### LODs generate nested subqueries
Nested LODs compound the cost. High-cardinality dimensions in LODs risk cross-joins.

### Table calculations run client-side
Never pushed to the database. Performance degrades with mark count.

### CASE is faster than IF/ELSEIF
ELSEIF (one word) is faster than ELSE IF (two words — creates nested statements).

### MIN/MAX perform better than AVG/ATTR

### Custom SQL is usually an anti-pattern
Wraps everything as subquery, disables join culling and query fusion.
Use native tables or database views instead.

---

## Reference files

| File | Load when... |
|------|-------------|
| `references/performance.md` | Dashboard optimization, extract strategies, filter tuning, server caching, calculated field anti-patterns |
| `references/twbx-parsing.md` | TWB/TWBX XML structure, datasource detection, calculated field extraction, Custom SQL audit, programmatic analysis |
| `references/migration.md` | Tableau-to-Power BI migration, DAX mappings, complexity scoring, feature parity gaps, data model translation |
| `references/api-governance.md` | REST API, Metadata API (GraphQL), Python TSC library, governance automation, lineage queries |
