# PBIX/PBIT Structure & TMDL

## PBIX file structure (ZIP archive)
- `Report/Layout` — report layout JSON (UTF-16 LE, no extension)
- `DataModel` — VertiPaq compressed binary (XPress9 → ABF)
- `DataMashup` — Power Query M code (nested ZIP)
- `DiagramLayout`, `Metadata`, `Settings`, `Connections` — JSON
- `SecurityBindings` — RLS config (binary)

DataModel binary layers: ZIP → XPress9 → ABF → VertiPaq column store.
Inside ABF: `metadata.sqlitedb` (all model metadata as SQLite).

## PBIT: most parseable format
Same ZIP as PBIX but `DataModelSchema` (readable JSON/TMSL) instead of `DataModel`.
Complete model definition without data. 21 MB PBIX → 61 KB PBIT.

## Layout JSON
UTF-16 LE encoded. `config`, `filters`, `query`, `dataTransforms` are double-serialized
(JSON strings within JSON). Must parse twice.

Visual type identifiers: columnChart, barChart, lineChart, areaChart, pieChart, donutChart,
tableEx, pivotTable, card, multiRowCard, slicer, map, filledMap, treemap, waterfallChart,
funnel, gauge, kpi, textbox, image, shape, actionButton, bookmarkNavigator.

## Modifying PBIX via ZIP
What works: in-place Layout replacement (open ZIP, replace Report/Layout).
What breaks: extract-all + re-zip (XPress9 compression ≠ standard deflate = corrupt files).

## TMDL format (GA 2024)
Human-readable, YAML-like text for semantic model definitions.
Default format for PBIP and Fabric Git integration.

```
table Sales
    lineageTag: e9374b9a-...

    measure 'Sales Amount' = SUMX('Sales', 'Sales'[Quantity] * 'Sales'[Net Price])
        formatString: $ #,##0
        displayFolder: Revenue

    column 'Product Key'
        dataType: int64
        isHidden
        sourceColumn: ProductKey
```

Folder structure:
```
definition/
├── tables/Sales.tmdl, Product.tmdl, Calendar.tmdl
├── roles/role1.tmdl
├── relationships.tmdl
├── model.tmdl
├── database.tmdl
```

## pbi-tools (MIT, open source)
- Extract PBIX → source files: `pbi-tools extract report.pbix -modelSerialization Tmdl`
- Compile → PBIT (cross-platform): `pbi-tools.core compile ./src -format PBIT -outPath out.pbit`
- Deploy: `pbi-tools deploy . Profile -environment Production`
- Generate BIM: `pbi-tools generate-bim -folder ./src`
- Cannot compile full PBIX with data (only PBIT)

## PBIXRay (Python, read-only)
```python
from pbixray import PBIXRay
model = PBIXRay("report.pbix")
model.tables          # list tables
model.measures        # all measures with DAX
model.power_query     # M code
df = model.get_table("Sales")  # table data as DataFrame
```

## Tools landscape
| Tool | Key capability |
|------|---------------|
| pbi-tools | Extract/compile/deploy, CI/CD |
| Tabular Editor 2 (free) | TOM editing, C# scripting, BPA, CLI |
| Tabular Editor 3 (paid) | + DAX IntelliSense, debugger, diagram |
| XMLA endpoints | Read/write TOM, DAX queries, granular refresh |
| PBIXRay | Parse VertiPaq, extract schema/measures |
| ALM Toolkit | Schema comparison, metadata deploy |
