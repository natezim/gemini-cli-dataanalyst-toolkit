---
name: dataprep
description: Google Cloud Dataprep / Trifacta Wrangle knowledge — DSL operators, flow export structure, transpilation to pandas/PySpark/BigQuery SQL, type-mismatch risks, and migration playbook. Activate when the user mentions Dataprep, Trifacta, Wrangle recipes, flow.json, or migrating off Dataprep.
---

# Dataprep / Trifacta Wrangle

Google Cloud Dataprep (Trifacta Classic) is a visual data engineering platform. Recipes are authored in **Wrangle**, a declarative DSL, then compiled at runtime to either Dataflow (Apache Beam) or BigQuery Standard SQL based on the source.

This skill covers what you need to read, understand, and migrate Wrangle pipelines into modern Python or SQL.

## When this skill activates

- User mentions: Dataprep, Trifacta, Wrangle, `flow.json`, `recipes/`, Dataprep migration
- User wants to translate a Wrangle recipe to Python (pandas/PySpark) or BigQuery SQL
- User exported a flow ZIP and wants to know what's in it

## Companion agent

`@dataprep-migrator` — read-only agent that translates Wrangle to pandas / PySpark / BigQuery SQL. Use it for the actual conversion. This skill provides the reference material it (and you) need.

## Core concepts (quick)

- **Wrangle** — declarative DSL. Doesn't execute on its own; the Dataprep engine compiles it to Beam (Dataflow) or SQL (BigQuery) at runtime.
- **Recipe** — sequential list of Wrangle steps.
- **Flow** — DAG of recipes, datasets, and outputs.
- **Flow node** — every dataset, recipe, and output in the flow has an immutable node ID.
- **Flow edge** — directional link between two flow nodes (`inputFlownode.id` → `outputFlownode.id`).
- **Output object** — defines where compiled output lands (CSV, JSON, Avro, Parquet) + ingestion mode (New/Update/Truncate/Load).
- **Sampling** — design-time only. Random/stratified/cluster/anomaly samples up to 10MB or first N rows. Production runs use full data.

## Reference files

- `references/wrangle-language.md` — full Wrangle operator registry with pandas / PySpark / BigQuery SQL equivalents
- `references/export-mechanics.md` — REST API endpoints, `trifacta` Python SDK usage, flow package ZIP structure
- `references/migration-risks.md` — temporal string drift, decimal precision truncation, null propagation discrepancies (the three risk areas that bite migrations)
- `references/validation.md` — 4-tier validation framework: schema → row count → MD5 checksum → cell-level diff
- `references/playbook.md` — phased migration execution (Discovery → Compilation → Validation → Cutover)

## Targets — which to choose

| If source data is... | And you want... | Pick |
|---|---|---|
| In BigQuery | Pure SQL, no infra | **BigQuery SQL** (cleanest replacement — Dataprep compiles to BQ SQL natively when source is BQ) |
| Big (>>1GB), distributed | Replace Dataflow | **PySpark** on Dataproc / Databricks / open-source Spark |
| Small/medium, single machine, notebook work | Local Python | **pandas** |

## What this skill is NOT

- Not a guide to using the Dataprep UI (Alteryx docs cover that)
- Not a wrapper around `trifacta` SDK (use the SDK directly; see `export-mechanics.md` for examples)
- Not a substitute for `@dataprep-migrator` — this skill is knowledge; the agent does translation
