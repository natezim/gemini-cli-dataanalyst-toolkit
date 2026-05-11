# REST API, XMLA & Deployment

## REST API base: `https://api.powerbi.com/v1.0/myorg/`
OAuth 2.0 bearer token, audience: `https://analysis.windows.net/powerbi/api`

## Scanner API — tenant-wide metadata (no Premium required)
Returns all datasets, tables, columns, measures with DAX, M queries, lineage.
Requires admin role. 100 workspaces per call, 500 req/hour.
```
POST .../admin/workspaces/getInfo?lineage=True&datasourceDetails=True&datasetSchema=True&datasetExpressions=True
Body: {"workspaces": ["workspace-guid"]}
```
Poll scanStatus, then get scanResult. Enable tenant settings for DAX/mashup expressions.

## Execute DAX queries (Pro license, no Premium)
```
POST .../datasets/{datasetId}/executeQueries
{"queries": [{"query": "EVALUATE SUMMARIZECOLUMNS(...)"}]}
```
1 query/request, 100K rows max, 120 req/min/user, 120s timeout.
Requires Dataset.Read.All + Build permission.

## XMLA endpoints — most powerful programmatic interface
Connection: `powerbi://api.powerbi.com/v1.0/myorg/{Workspace}`
Available on ALL Fabric SKUs (F2+) since June 2025.

Capabilities: read/write TOM, DAX queries, granular table/partition refresh,
deploy models, manage calculation groups, Object Level Security.

C# via TOM:
```csharp
var server = new TOM.Server();
server.Connect("DataSource=powerbi://api.powerbi.com/v1.0/myorg/Workspace");
foreach (var table in server.Databases[0].Model.Tables)
    foreach (var measure in table.Measures)
        Console.WriteLine($"{measure.Name} = {measure.Expression}");
```

## Tabular Editor CLI
```bash
# Validate with BPA
TabularEditor.exe "Model.bim" -A "BPARules.json" -V -E -W

# Deploy via XMLA
TabularEditor.exe "Model.bim" -D "powerbi://api.powerbi.com/v1.0/myorg/Workspace" "Model" -O -C -P -S -R -M

# Load TMDL, run script, deploy
TabularEditor.exe "definition/" -S "script.csx" -D "connection" "Model" -O -C -P -S
```
Flags: -O overwrite, -C connection swap, -P partitions, -S structure, -R roles, -M members.

## Best Practice Analyzer rules (JSON)
```json
{
  "ID": "PERF_AVOID_BIDIR",
  "Name": "Avoid bidirectional relationships",
  "Category": "Performance",
  "Severity": 3,
  "Scope": "Relationship",
  "Expression": "CrossFilteringBehavior == CrossFilteringBehavior.BothDirections",
  "FixExpression": "CrossFilteringBehavior = CrossFilteringBehavior.OneDirection"
}
```
Severity 3+ = error (stops pipeline). Run via CLI with -A flag.

## Enhanced refresh (Premium/Fabric)
Table/partition-level, bypasses 48/day limit:
```
POST .../groups/{id}/datasets/{id}/refreshes
{"type": "Full", "commitMode": "transactional", "objects": [{"table": "Sales"}]}
```

## Deployment pipelines API
```
POST .../pipelines/{id}/deployAll
{"sourceStageOrder": 0, "options": {"allowOverwriteArtifact": true}}
```
Stages: 0=Dev, 1=Test, 2=Prod. Requires Premium/Fabric.

## SemPy (Fabric notebooks)
```python
import sempy.fabric as fabric
fabric.list_datasets()
fabric.list_measures("MyDataset")
fabric.evaluate_dax("MyDataset", "EVALUATE SUMMARIZECOLUMNS(...)")
df = fabric.read_table("MyDataset", "Customer")
```

## CI/CD pipeline (GitHub Actions)
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    container: ghcr.io/pbi-tools/pbi-tools-core:latest
    steps:
      - uses: actions/checkout@v3
      - run: pbi-tools compile ./src -format PBIT -outPath ./out/Report.pbit -overwrite
      - run: pbi-tools deploy . Profile -environment Production
```
