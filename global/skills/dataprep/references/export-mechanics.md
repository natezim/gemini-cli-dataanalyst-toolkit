# Exporting Dataprep Flows — API, SDK, and Package Structure

## Flow package export (REST API)

Get the full package as a ZIP:

```
GET /v4/flows/{id}/package
Authorization: Bearer <API_TOKEN>
```

Response: a ZIP containing JSON descriptors for the entire flow.

### ZIP structure

```
flow-{id}-package/
├── flow.json                  Master metadata: node IDs, ownership, env params, global config
├── recipes/                   One JSON per recipe — sequential Wrangle DSL steps
│   ├── recipe-{id1}.json
│   └── recipe-{id2}.json
├── inputs/                    Upstream data source configs (BigQuery tables, GCS URIs, DB connections)
├── outputs/                   Publishing destinations (CSV/JSON/Avro/Parquet) + ingestion mode
├── webhooks/                  Event-driven hook definitions
└── *.data                     Binary/CSV mapping files for TBE (Transformation by Example) and manual cluster overrides
```

## Understanding the DAG

The flow is a Directed Acyclic Graph. Two key concepts:

- **Flow nodes** — every dataset, recipe, and output has an immutable node ID
- **Flow edges** — directional: `inputFlownode.id` → `outputFlownode.id`

### Walking the DAG programmatically

```python
# 1. Inventory all flows
GET /v4/flows

# 2. Get a specific flow's nodes
GET /v4/flows/{id}?embed=flownodes
# Returns: array of nodes, each with nested recipe.id, dataset.id, output.id

# 3. Get the edges
GET /v4/flows/{id}?embed=flowEdges
# Returns: array of edges with inputFlownode.id and outputFlownode.id

# 4. For each input dataset, find its parsing recipe and locate the downstream node
GET /v4/flows/{id}/inputs
# Match dataset.parsingRecipe.id against flownodes[].recipe.id
# That gives you the input node ID
# Then find the edge where inputFlownode.id matches → outputFlownode.id is the downstream

# 5. Swap sources programmatically (for dev → prod cutover)
POST /v4/flows/{id}/replaceDataset
POST /v4/flows/{id}/replaceInputDataset
```

## Native Python script generation

Dataprep can compile a recipe directly to a Python script via API:

```
POST /trifactaClassic/v1/outputObjects/{outputId}/generatePythonScript
Authorization: Bearer <API_TOKEN>
```

Response: JSON with the Python script as a plain-text string.

**Prerequisites:**
- Workspace admin must enable **"Wrangle to Python Conversion"** feature (disabled by default)
- Output object must exist (the script generates from an output, not a raw recipe)

**Limits — the SDK / API DOES NOT handle:**
- Nested types: `MapType`, `ArrayType`
- Non-CSV input formats: Excel, Avro, Parquet, JSON often fail
- Specific functions: `NUMFORMAT`, some custom string comparison functions

When these come up, fall back to manual translation (see `wrangle-language.md`) or the `@dataprep-migrator` agent.

## Python SDK usage

The `trifacta` package (PyPI) wraps the API for notebook workflows:

```python
import pandas as pd
import trifacta as tf

# Connect to your Dataprep workspace
client = tf.Client('https://clouddataprep.com', 'API_ACCESS_TOKEN')

# Reference an existing flow by ID
wrangle_flow = tf.wrangle_existing(flow_id=1432)

# Generate pandas code for a specific recipe
pandas_code = wrangle_flow.get_pandas(
    add_to_next_cell=False,
    recipe_name='Candidate Master Clean'
)

# Save to a script file
with open('generated_pipeline.py', 'w') as f:
    f.write(pandas_code)
```

**Same limits apply.** The SDK is a thin wrapper over the API.

## Mapping the package back to a project structure

For each recipe in the export:

| Wrangle thing | Where it goes in your new project |
|---|---|
| `flow.json` metadata | `./context/dataprep-flow-<id>.json` (READ-ONLY reference) |
| Each `recipes/*.json` Wrangle steps | `./output/queries/<recipe>.sql` or `./output/code/<recipe>.py` |
| `inputs/` source configs | `./.env.example` (variable names) + `./CONTEXT.md` (system inventory) |
| `outputs/` destinations | Orchestrator config (Cloud Composer DAG, dbt model, etc.) |
| `*.data` TBE mappings | `./output/data/<recipe>-mappings.csv` + handler script |
| `webhooks/` | Orchestrator notification config |

## What to inventory before starting a migration

Run these queries first via API to know what you're dealing with:

1. Total flows: `GET /v4/flows` — count
2. Per flow: recipe count, input count, output count
3. Connection inventory: all unique databases, tables, GCS paths referenced
4. Schedule inventory: which flows run on what cadence
5. Webhook inventory: which downstream systems get notified

That's Phase I (Discovery) of the migration playbook (`references/playbook.md`).
