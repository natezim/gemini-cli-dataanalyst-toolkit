# Gemini CLI — Data Analyst Toolkit

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup for data analysts, data engineers, and data scientists. Audit trail, controlled versioning, production safety, SQL query management, and skills for Python, BigQuery, and Excel.

> See the [full guide](GUIDE.md) for detailed usage.

## Who This Is For

Anyone doing data work with Gemini CLI who wants:
- **Audit trail** — every write, exec, and query logged with user identity for traceability
- **Controlled versioning** — auto-snapshots files at session end (max 3 versions, rotating)
- **File discipline** — strict folder structure, scratch sprawl prevented
- **Smart session management** — `/start` resumes where you left off, `/session:save` cleans up
- **SQL query library** — queries saved, tested, executions logged with cost
- **Production safety** — read-only by default, `.env` never read, DDL needs explicit approval
- **Chat log capture** — your prompts saved at session end (your words, not Gemini's)
- **Domain skills** — Python, BigQuery, and Excel expertise that activates automatically

Works for SQL analysts, Python data scientists, data engineers, ML engineers, and anyone iterating on data with Gemini CLI.

## Quick Start

1. Copy `starter-kit/` contents into your project root.
2. Copy any skills from `skills/` into `.gemini/skills/`.
3. Drop reference files (schemas, docs, specs) into `context/`.
4. Run `/start`.

First time? It scans your project, asks smart questions, builds your context.
Returning? Loads everything, resumes from your last session.

## What's Inside

```
GEMINI.md              Core rules — auto-loaded
rules/rules.md         Detail layer — safety, queries, audit codes
CONTEXT.md             Your project context (built by /start)
PREFERENCES.md         Your preferences + workflow overrides
.env.example           Connection variable names (Gemini reads this, not .env)
.gitignore             Protects credentials and outputs from git
.geminiignore          Keeps secrets and PII out of context

context/               Reference files — READ-ONLY, auto-loaded

.gemini/
  settings.json        Checkpointing, compression, session retention
  commands/            /start, /setup, /session:save, /context:update,
                       /version, /info
  skills/              Skill folders that activate automatically

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

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session — captures user identity, snapshots file state |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — versioning, cleanup review, chat log, handoff doc |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 (rotating) |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/info` | Show available commands and features |
| `/memory add` | Persist a fact across sessions |
| `/restore` | Undo a file change (Gemini built-in) |

## Audit & Compliance Features

**Audit log** — every write, exec, and query writes a structured entry to `output/audit-log.md`:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

**User identity** — captured at session start from the working directory path. Used in every audit entry.

**Auto versioning** — at session end, modified files in `output/code/`, `output/reports/`, `output/queries/` get snapshotted as `_v1`, `_v2`, `_v3`. Rotates oldest out. Live working file keeps base name.

**Temp folder** — `output/temp/` is for throwaway test/debug files. Auto-wiped at every session end (no review prompt). Use it freely for experimentation.

**Chat log** — substantive user prompts saved to `output/prompts/<date>_prompts.md` at session end. Filters out short confirmations.

**Cleanup review** — at session end, Gemini lists files created this session and asks: keep all, delete all, or review one-by-one. No mid-session interruptions.

## Security Defaults

- `.env` files are NEVER read — Gemini reads `.env.example` for variable names only
- Project root is OFF LIMITS — no scratch/, temp/, or ad-hoc folders
- `context/` is READ-ONLY — originals never modified
- Credentials masked as `[REDACTED]` in any output
- No PII or raw data in logs — structure only (counts, column names, status)
- No data exfiltration via web fetches or external services without explicit user request
- `PREFERENCES.md` cannot override security rules

## Skills

Copy from `skills/` into your project's `.gemini/skills/`. Skills activate automatically based on what you're working on.

| Skill | What it knows |
|---|---|
| **python** | pandas/NumPy, scikit-learn, visualization (matplotlib/Plotly/Altair), testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI, gcloud storage |
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA, Office Scripts |

Want more? Drop your own skill folder into `.gemini/skills/` (each skill is a `SKILL.md` plus optional `references/`). See the [Gemini CLI skills docs](https://geminicli.com/docs/cli/skills/).

## Platform

Built for Windows environments. Works in cmd and PowerShell.

## License

[MIT](LICENSE)
