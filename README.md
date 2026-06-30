# Gemini CLI — Data Analyst Toolkit

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup for data analysts, engineers, and scientists. **14 specialized subagents**, **8 domain skills** (Python, BigQuery, Excel, Power BI, Tableau Server / Optimizer / BigQuery integration, Dataprep), audit trail, smart session-resume, production safety.

> See the [full guide](GUIDE.md) for usage details and the [agent cheatsheet](AGENTS.md) for which subagent to invoke when.

## Prerequisites

- **[Gemini CLI](https://github.com/google-gemini/gemini-cli)** installed
- **Git** (optional) — only required if you want Gemini's built-in `/restore` (file-change undo via checkpointing). The kit ships with checkpointing **disabled by default** so it works without git. Our `/session:save` versioning (`_v1`/`_v2`/`_v3`) covers the common undo cases without needing git.

To enable checkpointing later (requires git):
```
edit ~/.gemini/settings.json → general.checkpointing.enabled = true
```

## Install — two folders, two destinations

The kit ships as two folders that go to different places:

| Folder | Destination | When |
|---|---|---|
| `global/` | `~/.gemini/` | **Once per machine** — pick which agents/skills you want, copy them here |
| `project-template/` | `<your-project>/` | **Per new project** — copy the template into each project root |

Everything in `global/` is **modular**. Take what's useful, skip the rest.

### Install everything

```powershell
# Windows
Copy-Item -Recurse global\* "$env:USERPROFILE\.gemini\"
```

```bash
# macOS/Linux
cp -r global/* ~/.gemini/
```

### Install by bundle (pick the bundles you use)

**Core** (required — rules, commands, settings, base agents):

```powershell
$gem = "$env:USERPROFILE\.gemini"
Copy-Item global\GEMINI.md, global\PREFERENCES.md, global\settings.json $gem
Copy-Item -Recurse global\commands $gem
# Base agents — useful for any data work
Copy-Item global\agents\solution-designer.md, global\agents\query-validator.md, global\agents\schema-explorer.md, global\agents\data-profiler.md, global\agents\report-drafter.md $gem\agents\
```

**Stats / SAS work** (add to Core):

```powershell
Copy-Item global\agents\stats-advisor.md, global\agents\sas-migrator.md "$env:USERPROFILE\.gemini\agents\"
```

**Python / BigQuery / Excel** (most common):

```powershell
Copy-Item -Recurse global\skills\python, global\skills\bigquery, global\skills\excel "$env:USERPROFILE\.gemini\skills\"
Copy-Item global\agents\excel-reviewer.md, global\agents\powerquery-reviewer.md "$env:USERPROFILE\.gemini\agents\"
```

**Power BI** (add when you work in Power BI):

```powershell
Copy-Item -Recurse global\skills\powerbi "$env:USERPROFILE\.gemini\skills\"
Copy-Item global\agents\dax-reviewer.md "$env:USERPROFILE\.gemini\agents\"
```

**Tableau** (add when you work in Tableau):

```powershell
Copy-Item -Recurse global\skills\tableau-server, global\skills\tableau-optimizer, global\skills\tableau-bigquery "$env:USERPROFILE\.gemini\skills\"
Copy-Item global\agents\tableau-auditor.md, global\agents\viz-optimizer.md, global\agents\migration-mapper.md "$env:USERPROFILE\.gemini\agents\"
```

### Per-project setup

In each project root:

```powershell
Copy-Item -Recurse <kit-repo>\project-template\* .
```

Then run `/start` in Gemini CLI. It identifies you, scans the project, asks only what it can't infer, and builds out `CONTEXT.md`.

### Updates

Pull the latest from this repo, copy the parts you have installed into `~/.gemini/` again. Existing projects' `CONTEXT.md` and `MEMORY.md` are untouched.

## Who This Is For

Anyone doing data work with Gemini CLI who wants:

- **14 specialized subagents** running in isolated context windows so heavy work doesn't bloat the main session
- **8 domain skills** — Excel, Python, BigQuery, Power BI, Tableau (server / optimizer / BigQuery), Dataprep / Wrangle migration
- **Audit trail** — every write, exec, and query logged with user identity
- **Smart session resume** — come back in 2 months, `/start` highlights stale threads, agent picks up without re-explaining
- **Controlled versioning** — auto-snapshots files at session end (max 3, rotating)
- **File discipline** — strict folder structure
- **Plan Mode + gated execution** — Pro researches, Flash implements, you approve risky work
- **Production safety** — read-only by default, `.env` never read, DDL needs explicit approval
- **3-tier memory with recurrence gate** — agent persists learnings, but only after they re-appear across sessions (prevents fabricated facts polluting permanent memory)

## What's in the Kit

### `global/` (installs to `~/.gemini/`)

```
GEMINI.md              Global rules — file discipline, audit, security, etc.
PREFERENCES.md         User preferences template
settings.json          Checkpointing, Plan Mode, compression, file auto-load list
commands/              /start, /session:save, /version, /info
agents/                14 specialized subagents (pick which to install)
skills/                8 domain skills (pick which to install)
```

### `project-template/` (copy into each new project)

```
CONTEXT.md             Project description (you write or /start helps you write)
MEMORY.md              Active threads + Learned (agent-managed, you prune)
.gitignore             Sensible defaults
.geminiignore          What Gemini shouldn't see (secrets only, not its own work)
.env.example           Connection variable names template
context/               Drop schemas, docs, specs here (READ-ONLY, auto-loaded)
output/                Workspace
  queries/             SQL library — one .sql per query
  code/                Scripts (.py, .js, .sh, .ipynb, configs)
  reports/             Analyses, docs (.md)
  data/                Exports (.csv, .json, .xlsx, .parquet)
  temp/                Throwaway test/debug — auto-wiped at session end
```

## Subagents (14 total)

Invoke explicitly with `@name`. Read-only agents are parallel-safe.
See [AGENTS.md](AGENTS.md) for the decision tree.

| Agent | Role | Mode |
|---|---|---|
| `@solution-designer` | Brainstorm, decompose, weigh tradeoffs | read-only |
| `@query-validator` | SQL syntax + dry-run cost check | read-only |
| `@schema-explorer` | Schema Q&A from context/ without DDL dump | read-only |
| `@data-profiler` | Shape, nulls, cardinality | read-only |
| `@stats-advisor` | Statistical method + assumptions + library calls | read-only |
| `@sas-migrator` | SAS DATA/PROC/macros → Python | read-only |
| `@dataprep-migrator` | Dataprep/Trifacta Wrangle → pandas / PySpark / BigQuery SQL | read-only |
| `@excel-reviewer` | Formulas, dynamic arrays, Power Pivot | read-only |
| `@powerquery-reviewer` | M language review, query folding | read-only |
| `@dax-reviewer` | DAX measure correctness + perf | read-only |
| `@tableau-auditor` | Tableau Server inventory, usage, permissions | read-only |
| `@viz-optimizer` | TWBX/PBIP performance issues, ranked | read-only |
| `@migration-mapper` | Tableau ↔ Power BI migration plan | read-only |
| `@report-drafter` | Markdown reports to `output/reports/` | write-scoped |

Built-ins: `@generalist`, `@codebase_investigator`, `@cli_help`.

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume — captures user identity, reads handoff, highlights stale threads |
| `/session:save` | End session — versioning, MEMORY.md update, handoff diff |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 |
| `/info` | Show available commands, agents, layout |
| `/memory add <fact>` | (built-in) Persist a global fact in `~/.gemini/GEMINI.md` |
| `/memory refresh` | (built-in) Re-read GEMINI/CONTEXT/MEMORY after editing |
| `/restore` | (built-in) Undo a file change — *requires checkpointing enabled (off by default; needs git)* |
| `@<agent>` | Invoke a specialized subagent |

## How session resume works

Gemini hierarchically auto-loads three files into context:

1. `~/.gemini/GEMINI.md` — global rules
2. `<project>/CONTEXT.md` — your project description
3. `<project>/MEMORY.md` — Active threads + Learned facts

Plus `/start` reads the most recent `output/<date>_handoff.md` to see what changed last session.

**Recurrence gate (prevents hallucinated memory):**
- Session 1: agent observes a maybe-fact → stages in handoff's `## Proposed learnings`
- Session 2: agent observes the SAME fact again → records in this handoff's proposed section too
- Session 3 `/start`: sees the fact in TWO consecutive handoffs → promotes to `MEMORY.md`
- Single-occurrence speculation never makes it to permanent memory

After a 2-month gap: `/start` says *"Welcome back. 3 open threads, one stale. 2 facts re-confirmed across sessions — promoting. What are we working on?"*

## Audit & Compliance

State-changing actions write a structured line to `output/audit-log.md`:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

User identity is captured from the working directory path at session start. Reads, list_directory, and session boundaries are NOT logged.

## Security Defaults

- `.env` files are NEVER read — Gemini reads `.env.example` only
- Project root is OFF LIMITS for new files
- `context/` is READ-ONLY
- Credentials masked as `[REDACTED]` in output
- No PII or raw data in logs — structure only
- No data exfiltration via web fetches without explicit user request
- Each subagent has an explicit `tools:` allowlist (least privilege)
- `PREFERENCES.md` cannot override security rules

## Skills (8 total)

| Skill | What it knows |
|---|---|
| **python** | pandas/NumPy, scikit-learn, visualization, testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI |
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA, Office Scripts |
| **powerbi** | DAX patterns, semantic models, TMDL, XMLA, Tabular Editor, Fabric, CI/CD |
| **tableau-server** | PostgreSQL repository, RMT, jobs, permissions, auditing |
| **tableau-optimizer** | Dashboard performance, TWBX parsing, Power BI migration |
| **tableau-bigquery** | SQL push-down, cost control, BI Engine, materialized views |
| **dataprep** | Wrangle DSL operators, flow export structure, transpilation to pandas/PySpark/BigQuery SQL, type-mismatch risks, 4-tier validation, phased migration playbook |

## Platform

Built and tested on Windows (PowerShell). Works on macOS/Linux — adjust install commands as shown.

## License

[MIT](LICENSE)
