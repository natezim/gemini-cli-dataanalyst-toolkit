# Power Query (M)

## Core concepts
- Primary Excel-native ingestion + transformation technology
- M is functional, case-sensitive, with type system
- Repeatable transforms, many connectors, separation between raw source and curated table
- Query folding pushes transforms to source — filter early, select early

## Connector examples

CSV with explicit encoding:
```powerquery
let
    Source = File.Contents(SourcePath),
    Csv = Csv.Document(Source, [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    Promoted = Table.PromoteHeaders(Csv, [PromoteAllScalars=true])
in Promoted
```

ODBC (folding-friendly):
```powerquery
let
    Source = Odbc.DataSource(ConnString, [HierarchicalNavigation=true])
in Source
```

Web/API with auth:
```powerquery
let
    Response = Web.Contents(BaseUrl, [
        Headers=[Authorization="Bearer " & Token],
        Timeout=#duration(0,0,0,30)
    ]),
    Json = Json.Document(Response)
in Json
```

## Reshape primitives

**Unpivot** (wide → long):
```powerquery
Table.UnpivotOtherColumns(Source, {"ID", "Name"}, "Month", "Amount")
```

**Merge/join**:
```powerquery
Table.NestedJoin(Orders, {"CustID"}, Customers, {"ID"}, "Cust", JoinKind.LeftOuter)
```

**Fill down**:
```powerquery
Table.FillDown(Source, {"Category"})
```

**Text cleanup**:
```powerquery
Table.TransformColumns(Source, {
    {"Name", each Text.Trim(Text.Clean(_)), type text}
})
```

## Cleaning pipeline
```powerquery
let
    Source = Excel.CurrentWorkbook(){[Name="RawTable"]}[Content],
    Cleaned = Table.TransformColumns(Source, {
        {"Name", each Text.Trim(Text.Clean(_)), type text},
        {"Email", each Text.Trim(Text.Clean(_)), type text}
    }),
    Long = Table.UnpivotOtherColumns(Cleaned, {"ID", "Name", "Email"}, "Month", "Amount"),
    Filled = Table.FillDown(Long, {"Category"})
in Filled
```

## Query folding
- Folding translates M steps into source-native queries (SQL, OData, etc.)
- Folded = fast (server does the work). Unfolded = slow (all data pulled to Excel then filtered).
- Steps that commonly break folding: custom columns with M functions, merges on non-folded sources, type changes after certain transforms.
- Check: View Native Query in Power Query Editor. If grayed out, folding stopped.
- Filter early, select only needed columns early, minimize custom steps.

## Pitfalls
- Version differences across Desktop/Web/Mac — some transforms not available everywhere
- Parameterize with named ranges, not hardcoded paths
- Credential handling must be governed for shared workbooks
- Remove Duplicates permanently deletes — copy data first (Microsoft recommends this)
