# Fabric & 2024-2025 Features

## F-SKU pricing (P-SKUs being retired)

| F SKU | CUs | Max Dataset | Monthly |
|-------|-----|-------------|---------|
| F2 | 2 | 3 GB | ~$262 |
| F8 | 8 | 10 GB | ~$1,049 |
| F64 (=P1) | 64 | 25 GB | ~$4,995 |
| F128 (=P2) | 128 | 50 GB | ~$9,990 |

F64+ boundary: free viewer access, paginated reports, deployment pipelines.
F-SKUs support pause/resume and pay-as-you-go (P-SKUs cannot).

## XMLA on ALL SKUs (June 2025)
XMLA read/write available on F2+ by default. Previously required F64+.
Game-changer for programmatic access to semantic models.

## Direct Lake
Reads Delta/Parquet from OneLake without importing into VertiPaq.
Import performance + DirectQuery freshness (no refresh needed).

Two flavors: Direct Lake on SQL (original, DQ fallback) and Direct Lake on OneLake (new, multi-source).

Guardrails:
| | F64 | F128 | F256 |
|---|---|---|---|
| Max rows/table | 300M | 600M | 1.2B |
| Max model on disk | 10 GB | 20 GB | 40 GB |

DirectLakeBehavior: "Automatic" (fallback to DQ), "DirectLakeOnly" (error on fallback).
Requires compatibilityLevel 1604+.

## PBIP format (source-control-friendly)
Replaces binary .pbix for development. Folder structure with text files.
TMDL is default model format. Enhanced Report Format (PBIR, June 2024) breaks
report layer into individual JSON files per visual/page.

```
MyReport.pbip
MyReport.Report/report.json
MyReport.SemanticModel/
  model.tmdl
  tables/Sales.tmdl
  relationships.tmdl
```

## Git integration
Fabric workspaces connect to Azure DevOps or GitHub.
Since January 2025, exports semantic models as TMDL (not TMSL).
Supported: reports, semantic models, dataflows, notebooks, pipelines.

## Semantic models (renamed from datasets)
"Datasets" renamed November 2023. No functional changes.
Legacy REST API still uses `datasets` in paths.
New Fabric REST API uses `semanticModels`:
```
GET https://api.fabric.microsoft.com/v1/workspaces/{id}/semanticModels
POST .../semanticModels/{id}/getDefinition
POST .../semanticModels/{id}/updateDefinition
```

## TMDL GA (2024)
Requires AS client libraries version 19.84.6+.
TMDL view in Desktop: syntax highlighting for DAX/M, diff view, drag-and-drop.

## Copilot
Creates report pages, generates DAX measures, narrative visuals, chat with data.
Fabric Copilot Capacity (FCC) from F2+ (~$262/mo) since April 2025.
Full workspace Copilot: F64+ recommended.
Cannot: complex visuals, custom visuals, sovereign clouds.

## SemPy (semantic-link)
Microsoft's Python library for Fabric notebooks. Pre-installed in Runtime 1.2+.
```python
import sempy.fabric as fabric
fabric.list_datasets()
fabric.evaluate_dax("Dataset", "EVALUATE ...")
```
DAX cell magic: `%%dax "Dataset"` in notebook cells.
