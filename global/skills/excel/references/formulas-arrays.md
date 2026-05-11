# Formulas & Dynamic Arrays

## Structured references
Convert ranges to Tables. Use `=SUM(Sales[Revenue])` not `=SUM(B2:B100)`.
Converting Table back to range converts structured refs to absolute A1 refs.
Converting range to Table does NOT auto-rewrite existing formulas.

## Dynamic arrays (spill)
```excel
=FILTER(Table1, Table1[Status]="Active")     -- filtered subset
=UNIQUE(Table1[Customer])                     -- unique list
=SORT(Table1, Table1[Date], -1)               -- sorted descending
=SORTBY(data, sort_col, order)                -- sort by different column
=SUM(A2#)                                     -- spill reference operator
```
- Spill formulas NOT supported inside Excel Tables — place outside
- `#SPILL!` = output area blocked by existing data
- Spill references don't work to closed workbooks (`#REF!`)

## LET and LAMBDA
```excel
=LET(
  x, Table1[Amount],
  y, Table1[Region],
  SUM(FILTER(x, y="NE"))
)

=LET(
  cfg, LAMBDA(k, XLOOKUP(k, Config[Key], Config[Value])),
  cfg("TaxRate")
)
```
LET: define intermediate variables, evaluate once. Reduces complexity and recalculation.
LAMBDA: create reusable custom functions. Assign to named ranges for workbook-wide use.

## XLOOKUP (replaces VLOOKUP/INDEX-MATCH)
```excel
=XLOOKUP(A2, Customers[ID], Customers[Name], "Not found")
=XLOOKUP(A2, Customers[ID], Customers, , 0)  -- return entire row
```
Searches any direction, exact match default, inline error handling.

## Data validation
```excel
-- Custom formula: restrict to 0-100
=AND(ISNUMBER(A2), A2>=0, A2<=100)
```
Constrains inputs with lists, ranges, dates, custom formulas. Add input messages.
Complex rule sets need in-sheet documentation.

## Common pitfalls
- Dates: MM/DD vs DD/MM locale issues. Use TEXT(A2, "yyyy-mm-dd") for normalization.
- Numbers stored as text: breaks lookups, pivots, forecasting. Convert explicitly.
- Linked data types (Stocks, Geography) require online connectivity.
