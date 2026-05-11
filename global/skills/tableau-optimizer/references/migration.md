# Tableau to Power BI Migration

## What maps cleanly
Workbook → .pbix | Worksheet → Visual | Dashboard → Report page
Published Data Source → Shared semantic model | Extract → Import mode
Live connection → DirectQuery | Quick filters → Slicers
Highlight actions → Cross-highlight | Relative date filters → Relative date slicers
Top N filters → Top N filter pane | Viz-in-tooltip → Report page tooltips

## What requires work
- Dual axis → Combo chart (limited mark mixing)
- Reference lines → Analytics pane
- Parameters → What-If (numeric) or disconnected slicer tables (text)
- LOD expressions → CALCULATE/ALLEXCEPT patterns
- Table calculations → WINDOW/RANKX functions

## Migration blockers (no Power BI equivalent)
- Page shelf animation
- Set actions (complex DAX workarounds needed)
- Marks card flexibility (drag-drop encoding)
- Hierarchical layout containers
- Custom shapes on marks
- Multi-layer maps (needs ArcGIS/custom visuals)
- Parameter actions

## Critical architectural shift
Tableau uses flat, denormalized datasets. Power BI performs best with star schemas.
50-column denormalized table → decompose into fact + dimension tables.

- Custom SQL → Power Query M with native query folding (or `Value.NativeQuery()` for complex SQL)
- Data blending → composite models or proper star schema
- Power BI has no per-visual blending — data must be modeled with relationships
- Power BI supports only ONE active relationship between two tables (use USERELATIONSHIP for inactive)
- Tableau's join culling has no Power BI equivalent
- Microsoft recommends "Semantic Layer-First": build centralized models, then reports

## Key DAX mappings

LOD expressions:
- `{FIXED [Region] : SUM([Sales])}` → `CALCULATE(SUM(Sales[Amount]), ALLEXCEPT(Sales, Sales[Region]))`
- `{FIXED : SUM([Sales])}` → `CALCULATE(SUM(Sales[Amount]), ALL(Sales))`
- `{INCLUDE [Product] : SUM([Sales])}` + AVG → `AVERAGEX(SUMMARIZE(...), [ProdSales])`
- `{EXCLUDE [Cat] : SUM([Sales])}` → `CALCULATE(SUM(Sales[Amount]), REMOVEFILTERS(Sales[Cat]))`

Nested LOD decomposition:
- `{EXCLUDE [Month] : AVG({FIXED [Month] : SUM([Sales])})}` →
  Step 1: `MonthlySales = CALCULATE(SUM(Sales[Amount]), ALLEXCEPT(Sales, Sales[Month]))`
  Step 2: `AvgMonthlySales = AVERAGEX(VALUES(Sales[Month]), [MonthlySales])`

Table calculations:
- RUNNING_SUM → CALCULATE with FILTER/ALLSELECTED date range
- WINDOW_AVG → AVERAGEX with DATESINPERIOD
- RANK → RANKX(ALLSELECTED(...))
- INDEX() → RANKX with unique column or INDEX() (2023+ DAX)
- SIZE() → COUNTROWS(ALLSELECTED(table[dim]))
- TOTAL(SUM([Sales])) → CALCULATE(SUM(Sales[Amount]), ALLSELECTED(visible_dims))
- LOOKUP → CALCULATE with OFFSET (2023+ DAX)
- Percent of total: `SUM / TOTAL(SUM)` → `DIVIDE(SUM, CALCULATE(SUM, ALL))`

Function mapping:
- SUM → SUM | AVG → AVERAGE | COUNT → COUNTA | COUNTD → DISTINCTCOUNT
- MEDIAN → MEDIAN | ATTR → SELECTEDVALUE | MIN/MAX → MIN/MAX
- ZN([field]) → IF(ISBLANK(col), 0, col)
- ISNULL → ISBLANK | IFNULL → COALESCE
- CASE/WHEN/END → SWITCH(col, "a", 1, "b", 2)
- IF/ELSEIF/END → IF(cond, val, IF(cond, val, default))
- DATEADD('month', 3, [d]) → EDATE(col, 3)
- DATEDIFF: Tableau interval FIRST arg → DAX interval LAST arg
- DATETRUNC('month', [d]) → DATE(YEAR(col), MONTH(col), 1)
- DATEPART('year', [d]) → YEAR(col) | DATENAME('month', [d]) → FORMAT(col, "MMMM")
- CONTAINS → CONTAINSSTRING (DAX CONTAINS is a table function)
- REPLACE → SUBSTITUTE (DAX REPLACE is positional)
- FIND → SEARCH (case-insensitive) or FIND (case-sensitive)
- LEFT/RIGHT/MID/LEN/UPPER/LOWER/TRIM → identical in DAX
- No DAX: SPLIT, REGEXP_*, PROPER → need Power Query

Concept mappings:
- Sets → Calculated columns with RANKX + IF, or DAX FILTER in measures
- Parameters → What-If (numeric) or disconnected slicer table + SELECTEDVALUE (text)
- Groups → SWITCH(TRUE(), col="a", "Group1", ...) calculated column
- Bins → INT(col / binsize) * binsize calculated column

## 10 critical syntax differences
1. Tableau THEN/ELSE/END → DAX uses comma-separated args
2. DATEDIFF interval: first arg (Tableau) → last arg (DAX)
3. CONTAINS is string (Tableau) → table function (DAX) — use CONTAINSSTRING
4. REPLACE is pattern-based (Tableau) → positional (DAX) — use SUBSTITUTE
5. CASE/WHEN/END → SWITCH with comma-separated pairs
6. [field] references → table[column] references
7. Date literals use # (Tableau) → DATE() function (DAX)
8. Boolean TRUE/FALSE → TRUE()/FALSE() functions
9. Single quotes for date parts ('month') → enum constants (MONTH)
10. AVG → AVERAGE, COUNTD → DISTINCTCOUNT

## Complexity scoring

| Factor | Weight | Low (1) | Medium (2) | High (3) |
|---|---|---|---|---|
| Data sources | 3x | 1, extract | 2-3, mixed | 4+, cross-DB |
| Calculated fields | 2x | <10 simple | 10-30 nested | 30+ complex |
| LOD expressions | 4x | 0 | 1-5 FIXED | 5+ or nested |
| Table calculations | 3x | 0 | 1-5 default | 5+ custom addressing |
| Custom SQL | 2x | None | 1-2 simple | 3+ complex |
| Parameters | 2x | 0 | 1-3 numeric | 4+ or param actions |
| Dashboard actions | 3x | 0-2 filter | 3-5 mixed | 6+ or set/URL |
| Sets/set actions | 4x | None | 1-2 static | Dynamic + set actions |
| Worksheets/dashboards | 1x | <5 sheets | 5-15 sheets | 15+ sheets |
| Data blending | 3x | None | 1 blend | Multiple blends |

Score: ≤30 = Low (1-2 days) | 31-60 = Medium (3-5 days) | 61+ = High (5-15+ days)

## Per-calculation complexity scoring

| Factor | Weight | Low (1) | Medium (3) | High (5) |
|---|---|---|---|---|
| Nesting depth | 2x | 1 level | 2-3 levels | 4+ levels |
| COUNTD usage | 3x | None | Single | Multiple or in LOD |
| LOD complexity | 3x | None | Single FIXED | Nested or INCLUDE/EXCLUDE |
| Table calc scope | 2x | None | Simple RUNNING | Complex WINDOW + custom addressing |
| String operations | 1.5x | None | LEFT/RIGHT/MID | FIND/REPLACE/REGEX chains |
| Type conversions | 1.5x | None | 1 conversion | Chained conversions |
| Parameter in calc | 1x | None | Simple ref | Parameter in LOD/SQL |

1-10 = Simple (auto DAX) | 11-25 = Moderate (semi-auto) | 26-40 = Complex (manual) | 40+ = Redesign

## RLS translation
Tableau: user filters with USERNAME() as data source filters.
Power BI: DAX role definitions with USERPRINCIPALNAME(). Only applies to Viewer role.

## Governance mapping
- Tableau binary certification → Power BI three-tier endorsement (Promoted/Certified/Master Data)
- Tableau Data Quality Warnings → Microsoft Purview sensitivity labels
- Tableau published data sources (locally overridable) → Power BI shared semantic models (propagate changes automatically)

## Viz type mapping
Native equivalents: bar, column, line, scatter, treemap, waterfall.
Maps: Map/Filled Map/Azure Maps (limited multi-layer).
Manual setup: heat map, highlight tables (matrix + conditional formatting).
No native: Gantt, box plot, bullet, packed bubbles, histogram (need custom visuals).
Partial: dual axis → combo chart (limited).
