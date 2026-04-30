---
name: excel
description: >
  Universal Excel skill for formulas, Power Query, Power Pivot/DAX, data analysis,
  automation, and governance. Use when the user is: writing Excel formulas, building
  dynamic arrays (FILTER/UNIQUE/SORT), creating Power Query M transformations, working
  with PivotTables or Data Model/DAX, automating with VBA or Office Scripts, using
  Python in Excel, optimizing workbook performance, troubleshooting volatile functions,
  building dashboards, cleaning data, or integrating via Graph API/Open XML.
---

# Excel Expert

Production-grade Excel for analysis, modeling, automation, and governance.
Covers formulas, dynamic arrays, Power Query M, Power Pivot DAX, VBA, Office Scripts,
Python in Excel, Graph API, and workbook optimization.

## Critical rules — always apply

### Use Tables, not raw ranges
Convert data to Excel Tables for automatic growth, structured references (`Sales[Revenue]`),
and reliable PivotTable sources. Structured references are self-documenting.

### Dynamic arrays don't work inside Tables
Spill formulas must be placed outside Tables. Reference table columns but output elsewhere.
`#SPILL!` = spill area blocked.

### Volatile functions kill performance
NOW, TODAY, RANDBETWEEN, OFFSET, INDIRECT recalculate on every change.
Replace: OFFSET → INDEX, INDIRECT → CHOOSE. Minimize volatile function usage.

### Power Query: filter early, select early
Query folding pushes transforms to the data source. Filtering late or adding custom
steps often breaks folding — difference between seconds and minutes on large data.

### Measures over calculated columns (DAX)
DAX measures aggregate dynamically. Calculated columns store per-row and bloat the model.
Use measures for KPIs, calculated columns only for row-level context needed in slicers/filters.

### CALCULATE modifies filter context
```dax
Revenue % Total = DIVIDE(
  SUM(Sales[Amount]),
  CALCULATE(SUM(Sales[Amount]), REMOVEFILTERS(Sales[Channel]))
)
```
Understand filter context before writing CALCULATE expressions.

### Never auto-enable macros or trust settings
VBA macros are a security vector. Never suggest enabling macros without explaining risks
and requiring explicit user confirmation. Provide Office Scripts alternatives first.

### Python in Excel cannot read external files
`pandas.read_csv` / `pandas.read_excel` are NOT compatible. Data must come from
worksheet ranges or Power Query. Use `xl('TableName[#All]', headers=True)`.

### XLOOKUP over VLOOKUP
```excel
=XLOOKUP(A2, Customers[ID], Customers[Name], "Not found")
```
XLOOKUP searches any direction, returns exact match by default, handles errors inline.

---

## Reference files

| File | Load when... |
|------|-------------|
| `references/formulas-arrays.md` | Formulas, dynamic arrays, LAMBDA/LET, structured references, data validation |
| `references/power-query.md` | Power Query M, connectors, cleaning, unpivot, merge, query folding |
| `references/modeling-dax.md` | Power Pivot, Data Model, DAX measures, PivotTables, forecasting, Python in Excel |
| `references/automation.md` | VBA, Office Scripts, Graph API, Office Add-ins, Open XML, Power Automate |
| `references/performance-governance.md` | Calculation modes, volatile functions, MTR, protection, co-authoring, security |
