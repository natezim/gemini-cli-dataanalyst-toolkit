# Data Modeling, DAX & Analytics

## PivotTables
1. Convert range → Table
2. Insert PivotTable from Table
3. Add slicers + timelines for interactive filtering

Slicers have limitations in Excel for web. Source must be a Table for reliable refresh.

## Power Pivot / Data Model
In-workbook relational modeling with DAX. Multiple tables, relationships, reusable measures.

**Measures** (dynamic aggregation — use for KPIs):
```dax
Total Revenue := SUM(Sales[Amount])

Revenue % Total := DIVIDE(
    SUM(Sales[Amount]),
    CALCULATE(SUM(Sales[Amount]), REMOVEFILTERS(Sales[Channel]))
)

Revenue After Discount := SUMX(Sales, Sales[Amount] * (1 - Sales[DiscountPct]))
```

**Calculated columns** (stored per-row — use sparingly):
Only for values needed in slicers, filters, or row-level context. Bloats model.

**CALCULATE**: modifies filter context. Core of DAX.
**SUMX/AVERAGEX**: iterators, evaluate expression row-by-row then aggregate.
**FILTER**: returns filtered table. Avoid as CALCULATE filter arg when boolean works.

## Forecasting
- **Forecast Sheet** (Data → Forecast Sheet): requires timeline with consistent intervals
- **ETS functions**: `FORECAST.ETS`, `FORECAST.ETS.CONFINT`, `FORECAST.ETS.SEASONALITY`
- Tolerates ~30% missing data points
- Summarize/aggregate before forecasting for better accuracy

## Analysis ToolPak
Add-in — must be enabled. Regression, histograms, descriptive stats, t-tests, ANOVA.
Analyzes one worksheet at a time. Odd behavior when worksheets grouped.

## Solver
Add-in for optimization. Not available on mobile.

## Python in Excel
```excel
=PY("
import pandas as pd
df = xl('MyTable[#All]', headers=True)
summary = df.describe(include='all')
summary
")
```
- Runs in Microsoft cloud (Anaconda distribution)
- `xl()` bridge: accepts ranges, tables, named ranges
- `[#All]` specifier for full table including headers
- **Cannot** read external files (no `read_csv`/`read_excel`) — data from sheet or Power Query only
- Not available on iOS/Android (cells error on recalc)
- Recalc order: row-major across sheets

## Common pitfalls
- Confusing measures vs calculated columns (performance + correctness)
- Using FILTER inside CALCULATE when a simple boolean works
- Not understanding filter context → wrong CALCULATE results
- Dates stored as text → forecasting and PivotTable failures
