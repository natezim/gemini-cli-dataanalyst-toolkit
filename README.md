# Gemini CLI — Data Analyst Toolkit

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup for data analysts, data engineers, and data scientists. 9 specialized subagents, deep skills for Python / BigQuery / Excel, audit trail, controlled versioning, production safety.

> See the [full guide](GUIDE.md) for detailed usage and the [agent cheatsheet](AGENTS.md) for when to use which subagent.

## Who This Is For

Anyone doing data work with Gemini CLI who wants:

- **9 specialized subagents** — narrow specialists (`@query-validator`, `@stats-advisor`, `@solution-designer`, etc.) that run in isolated context windows so heavy work doesn't bloat the main session
- **Domain skills** — Python, BigQuery, Excel — activate automatically based on your work
- **Audit trail** — every write, exec, and query logged with user identity for traceability
- **Controlled versioning** — auto-snapshots files at session end (max 3 versions, rotating)
- **File discipline** — strict folder structure, scratch sprawl prevented
- **Smart session management** — `/start` resumes where you left off, `/session:save` cleans up
- **Plan Mode + model routing** — Pro for architecture, Flash for implementation
- **Production safety** — read-only by default, `.env` never read, DDL needs explicit approval
- **3-tier memory** — global (~/.gemini/GEMINI.md), team rules (./GEMINI.md), project context (./CONTEXT.md with agent-appended Learned section)
- **Chat log capture** — your prompts saved at session end (your words, not Gemini's)

Works for SQL analysts, Python data scientists, data engineers, ML engineers, SAS migrators, and anyone iterating on data with Gemini CLI.

## Quick Start

1. Copy `starter-kit/` contents into your project root.
2. Copy any skills from `skills/` into `.gemini/skills/`.
3. (All 9 subagents in `starter-kit/.gemini/agents/` come along automatically.)
4. Drop reference files (schemas, docs, specs) into `context/`.
5. Run `/start`.

First time? It scans your project, asks only what it can't infer, builds your context.
Returning? Loads everything, resumes from your last session.

## What's Inside

```
GEMINI.md              Core rules — auto-loaded
rules/rules.md         Detail layer — safety, queries, audit codes
CONTEXT.md             Your project context (built by /start)
PREFERENCES.md         Your preferences + agent + workflow overrides
.env.example           Connection variable names (Gemini reads this, not .env)
.gitignore             Protects credentials and outputs from git
.geminiignore          Keeps secrets and PII out of context

context/               Reference files — READ-ONLY, auto-loaded

.gemini/
  settings.json        Checkpointing, Plan Mode + model routing, compression
  commands/            /start, /setup, /session:save, /context:update,
                       /version, /info
  skills/              Skill folders that activate automatically
  agents/              9 specialized subagents (invoke with @name)

output/                Workspace — everything Gemini produces
  queries/             SQL query library — one .sql per query
  code/                Scripts (.py, .js, .sh, .ipynb, configs)
  reports/             Analysis, docs (.md)
  data/                Exports (.csv, .json, .xlsx, .parquet)
  temp/                Throwaway test/debug files — auto-wiped at session end
  audit-log.md         Structured audit trail (silent, only on state changes)
  prompts/             Your chat prompts saved per session
  <date>_handoff.md    Session wrap-up docs
```

## Subagents (9 total)

Invoke explicitly with `@name`. Read-only agents are parallel-safe.
See [AGENTS.md](AGENTS.md) for invocation examples and decision tree.

| Agent | Role | Mode |
|---|---|---|
| `@solution-designer` | Brainstorm, decompose, weigh tradeoffs before writing | read-only |
| `@query-validator` | SQL syntax + dry-run cost check | read-only |
| `@schema-explorer` | Schema Q&A from context/ without dumping DDL | read-only |
| `@data-profiler` | Shape, nulls, cardinality on a dataset | read-only |
| `@stats-advisor` | Statistical method recommendation, assumptions, library calls | read-only |
| `@sas-migrator` | SAS DATA/PROC/macros → Python (pandas/statsmodels) | read-only |
| `@excel-reviewer` | Formulas, dynamic arrays, Power Pivot review | read-only |
| `@powerquery-reviewer` | M language review (Excel + Power BI), query folding | read-only |
| `@report-drafter` | Writes markdown reports to `output/reports/` only | write-scoped |

Built-ins (always available): `@generalist`, `@codebase_investigator`, `@cli_help`.

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session — captures user identity, snapshots file state |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — versioning, temp wipe, chat log, handoff doc |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 (rotating) |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/info` | Show available commands, agents, and features |
| `/memory add` | Persist a fact across sessions (built-in) |
| `/restore` | Undo a file change (built-in) |
| `@<agent>` | Invoke a specialized subagent |
| `@./file` | Inject any file into context mid-session |

## Audit & Compliance Features

**Audit log** — state-changing actions write a structured entry to `output/audit-log.md`:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

Session boundaries are NOT logged. Read-only operations are NOT logged.

**User identity** — captured at session start from the working directory path.

**Auto versioning** — at session end, modified files in `output/code/`, `output/reports/`, `output/queries/` get snapshotted as `_v1`, `_v2`, `_v3`. Rotates oldest out.

**Temp folder** — `output/temp/` is for throwaway test/debug files. Auto-wiped at every session end.

**Chat log** — substantive user prompts saved to `output/prompts/<date>_prompts.md` at session end.

**No cleanup prompt** — `output/temp/` is auto-wiped at session end (silent). Files in `code/`, `reports/`, `queries/`, `data/` are deliverables by rule and are never asked about. Throwaway work goes in `temp/`, period.

## Security Defaults

- `.env` files are NEVER read — Gemini reads `.env.example` for variable names only
- Project root is OFF LIMITS — no scratch/, temp/, or ad-hoc folders
- `context/` is READ-ONLY — originals never modified
- Credentials masked as `[REDACTED]` in any output
- No PII or raw data in logs — structure only (counts, column names, status)
- No data exfiltration via web fetches or external services without explicit user request
- Each subagent has an explicit `tools:` allowlist (least privilege)
- `PREFERENCES.md` cannot override security rules

## Skills

Copy from `skills/` into your project's `.gemini/skills/`. Skills activate automatically.

| Skill | What it knows |
|---|---|
| **python** | pandas/NumPy, scikit-learn, visualization (matplotlib/Plotly/Altair), testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI, gcloud storage |
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA, Office Scripts |

Want more? Drop your own skill folder into `.gemini/skills/`. Each skill is a `SKILL.md` plus optional `references/`.

## Platform

Built for Windows environments. Works in cmd and PowerShell.

## License

[MIT](LICENSE)
