# DAX Patterns & Anti-Patterns

## Core patterns

**Always use VAR** — evaluated once, cached, ~50% faster:
```dax
Sales YoY Growth % =
VAR PY = CALCULATE([Sales], PARALLELPERIOD('Date'[Date], -12, MONTH))
RETURN DIVIDE([Sales] - PY, PY)
```

**DIVIDE over /** — safe division, returns BLANK on zero:
```dax
Margin % = DIVIDE([Profit], [Revenue])
```

**CALCULATE with boolean filters, not FILTER**:
```dax
-- GOOD: iterates unique values only
Red Sales = CALCULATE(SUM(Sales[Amount]), Sales[Color] = "Red")
-- BAD: iterates every row
Red Sales = CALCULATE(SUM(Sales[Amount]), FILTER(Sales, Sales[Color] = "Red"))
```

**SUM for single column, SUMX only for cross-column math**:
```dax
Total = SUM(Sales[Amount])                              -- single column
Revenue = SUMX(Sales, Sales[Qty] * Sales[UnitPrice])   -- cross-column
```
Never nest SUMX inside SUMX (Cartesian product explosion).

**COUNTROWS not COUNT. SELECTEDVALUE not HASONEVALUE+VALUES.**

## Time intelligence
```dax
Sales YTD = TOTALYTD([Sales], 'Date'[Date])
Sales PY = CALCULATE([Sales], SAMEPERIODLASTYEAR('Date'[Date]))
Sales MTD = TOTALMTD([Sales], 'Date'[Date])
Sales QTD = TOTALQTD([Sales], 'Date'[Date])
Rolling 3M = CALCULATE([Sales], DATESINPERIOD('Date'[Date], MAX('Date'[Date]), -3, MONTH))
```
Requires a proper Date table marked as date table with no gaps.

## Scoped bidirectional with CROSSFILTER
```dax
Countries Sold = CALCULATE(
    DISTINCTCOUNT(Customer[Country]),
    CROSSFILTER(Customer[CustomerCode], Sales[CustomerCode], BOTH)
)
```
Use this instead of permanent bidirectional relationships.

## Row-Level Security (dynamic)
```dax
-- Single role, scales to thousands of users
-- Table permission on SecurityTable:
SecurityTable[UserEmail] = USERPRINCIPALNAME()
```
Security table: UserEmail + Region columns. Relationship to dimension table.
RLS adds <100ms overhead when implemented correctly.
Only applies to Viewer role — Admins/Members/Contributors bypass.

## Calculation groups (2024+)
Replace repetitive time intelligence by creating a calculation group with items
like YTD, PY, MTD that apply to any measure. Avoids creating N × M measures.

## Anti-patterns
- Nesting SUMX (Cartesian explosion)
- FILTER on entire tables instead of boolean conditions
- Repeating CALCULATE expressions instead of using VAR
- `a / b - 1` returns -1 when both blank — use DIVIDE
- Calculated columns where measures would work (~90% should be measures)
- Bidirectional relationships as default (use single-direction + CROSSFILTER)
