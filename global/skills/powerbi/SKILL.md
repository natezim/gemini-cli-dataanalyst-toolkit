---
name: powerbi
description: >
  Universal Power BI skill for creating and optimizing reports, semantic models, and DAX.
  Use when the user is: writing DAX measures, designing star schemas, optimizing report
  performance, working with DirectQuery or composite models, parsing PBIX/PBIT files,
  using TMDL or Tabular Editor, deploying via XMLA or REST API, implementing RLS,
  configuring incremental refresh, working with Fabric/Direct Lake, building CI/CD
  pipelines for Power BI, or migrating from other BI tools.
---

# Power BI Expert

Production-grade Power BI development, optimization, and deployment. Covers DAX, semantic
models, report design, PBIX internals, REST API, XMLA, TMDL, Tabular Editor, Fabric, and CI/CD.

## Critical rules — always apply

### Star schema is mandatory
Always use star schema. Dimension tables → one-to-many → fact tables.
Denormalize snowflake dimensions. Use integer surrogate keys.
Proper star schema reduces model size by ~60% and improves queries by 10x+.

### ~90% of DAX should be measures, not calculated columns
Calculated columns are stored per row in VertiPaq. One client converted 30 calcs
to measures: 4 GB → 1.2 GB model, 45 min → 6 min refresh.

### Use CALCULATE with simple boolean filters, not FILTER
```dax
-- GOOD: iterates ~20 unique colors
CALCULATE(SUM(Sales[Amount]), Sales[Color] = "Red")

-- BAD: FILTER iterates every row (millions)
CALCULATE(SUM(Sales[Amount]), FILTER(Sales, Sales[Color] = "Red"))
```

### Always use VAR for intermediate results
Variables evaluated once and cached. Microsoft confirms ~50% faster.

### Use DIVIDE() instead of /
`a / b` errors on zero. DIVIDE returns BLANK.

### Keep visuals under 8-10 per page
Each visual = its own DAX query. 20 visuals → 8 = load time from 5s to <1s.
Slicers are worst offenders — each generates 2 queries. Add "Apply" buttons.

### Single-direction filtering by default
Use `CROSSFILTER()` in specific measures instead of permanent bidirectional relationships.
Bidirectional creates ambiguous paths and performance issues.

### COUNTROWS not COUNT, SELECTEDVALUE not HASONEVALUE+VALUES

### Never nest SUMX inside SUMX (Cartesian product explosion)

### XMLA read/write available on ALL Fabric SKUs (F2+) since June 2025

---

## Reference files

| File | Load when... |
|------|-------------|
| `references/dax-patterns.md` | Writing DAX, time intelligence, RLS, calculation groups, anti-patterns |
| `references/model-design.md` | Star schema, DirectQuery, composite models, incremental refresh, storage modes |
| `references/report-optimization.md` | Visual performance, query reduction, slicers, bookmarks, custom visuals |
| `references/pbix-tmdl.md` | PBIX/PBIT structure, Layout JSON, TMDL format, pbi-tools, programmatic manipulation |
| `references/api-deployment.md` | REST API, XMLA, Tabular Editor CLI, Scanner API, deployment pipelines, CI/CD |
| `references/fabric-features.md` | F-SKUs, Direct Lake, PBIP format, Git integration, Copilot, SemPy |
