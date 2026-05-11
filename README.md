# Gemini CLI — Data Analyst Toolkit

A plug-and-play [Gemini CLI](https://github.com/google-gemini/gemini-cli) setup for data analysts, engineers, and scientists. **9 specialized subagents**, deep skills for Python / BigQuery / Excel, audit trail, smart session-resume, production safety.

> See the [full guide](GUIDE.md) for usage details, [agent cheatsheet](AGENTS.md) for which subagent to invoke when.

## Install (v2 — two folders, two destinations)

The kit ships as two folders that go to different places:

| Folder | Destination | When |
|---|---|---|
| `global/` | `~/.gemini/` | **Once per machine** — install all rules, agents, skills here |
| `project-template/` | `<your-project>/` | **Per new project** — copy the template into each project root |

### One-time global install

```powershell
# Windows PowerShell
Copy-Item -Recurse global\* "$env:USERPROFILE\.gemini\"
```

```bash
# macOS/Linux
cp -r global/* ~/.gemini/
```

That installs `GEMINI.md` (rules), `PREFERENCES.md`, `settings.json`, all 9 custom agents, all 3 skills, and the custom slash commands.

### Per-project setup

```powershell
Copy-Item -Recurse <kit-repo>\project-template\* .
```

Then run `/start`. First time, scans the project and writes `CONTEXT.md`. After that, every `/start` resumes from your last session.

### Updates

Pull the latest, copy `global/*` to `~/.gemini/` again. All projects pick up new rules on next `/start`.

## Who This Is For

Anyone doing data work with Gemini CLI who wants:

- **9 specialized subagents** in isolated context windows
- **Domain skills** — Python, BigQuery, Excel
- **Audit trail** with user identity
- **Smart session resume** — come back in 2 months, agent picks up without re-explaining
- **Controlled versioning** — auto-snapshots files at session end (max 3, rotating)
- **File discipline** — strict folder structure
- **Plan Mode + gated execution** — Pro for architecture, Flash for implementation, approval on risky work
- **Production safety** — read-only by default, `.env` never read, DDL needs explicit approval
- **3-tier memory with recurrence gate** — agent persists learnings only after they re-appear across sessions

Works for SQL analysts, Python data scientists, data engineers, ML engineers, SAS migrators, and anyone iterating on data.

## What's in the Kit

### `global/` (installs to `~/.gemini/`)

```
GEMINI.md              Global rules
PREFERENCES.md         User preferences template
settings.json          Checkpointing, Plan Mode, file auto-load list
commands/              /start, /session:save, /version, /info
agents/                9 specialized subagents
skills/                3 domain skills (python, bigquery, excel)
```

### `project-template/` (copy into each new project)

```
CONTEXT.md             Project description (you write or /start helps you write)
MEMORY.md              Active threads + Learned (agent-managed, you prune)
.gitignore, .geminiignore, .env.example
context/               Drop schemas, docs, specs here (READ-ONLY, auto-loaded)
output/                Workspace — queries, code, reports, data, temp
```

## Subagents (9 total)

| Agent | Role | Mode |
|---|---|---|
| `@solution-designer` | Brainstorm, decompose, weigh tradeoffs | read-only |
| `@query-validator` | SQL syntax + dry-run cost check | read-only |
| `@schema-explorer` | Schema Q&A without DDL dump | read-only |
| `@data-profiler` | Shape, nulls, cardinality | read-only |
| `@stats-advisor` | Statistical method + assumptions + library calls | read-only |
| `@sas-migrator` | SAS DATA/PROC/macros → Python | read-only |
| `@excel-reviewer` | Formulas, dynamic arrays, Power Pivot | read-only |
| `@powerquery-reviewer` | M language review, query folding | read-only |
| `@report-drafter` | Markdown reports to `output/reports/` | write-scoped |

Built-ins: `@generalist`, `@codebase_investigator`, `@cli_help`.

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume |
| `/session:save` | End session — versioning, MEMORY.md update, handoff diff |
| `/version <file>` | Manual snapshot to v1/v2/v3 |
| `/info` | Show available commands, agents, layout |
| `/memory add <fact>` | (built-in) Global fact → `~/.gemini/GEMINI.md` |
| `/memory refresh` | (built-in) Re-load files after editing |
| `/restore` | (built-in) Undo a file change |
| `@<agent>` | Invoke a specialized subagent |

## How session resume works

Gemini auto-loads three files into context:

1. `~/.gemini/GEMINI.md` — global rules
2. `<project>/CONTEXT.md` — project description
3. `<project>/MEMORY.md` — Active threads + Learned facts

Plus `/start` reads the most recent `output/<date>_handoff.md`.

**Recurrence gate** (prevents fabricated memory): agent observes a maybe-fact → stages in handoff's `## Proposed learnings` → next `/start` promotes only if it still applies. Single-observation speculation never lands in permanent memory.

After a 2-month gap, `/start` says *"Welcome back. 3 open threads, one stale. 2 facts I observed last session look right — promoting. What are we working on?"*

## Audit & Compliance

State-changing actions write a line to `output/audit-log.md`:
```
[2026-04-15 14:23:01] | jsmith | WRITE | output/code/analysis.py | OK
[2026-04-15 14:23:45] | jsmith | QUERY | bigquery:project.dataset.table | 142 rows | OK
```

Reads and session boundaries are NOT logged.

## Security Defaults

- `.env` files NEVER read; `.env.example` instead
- Project root OFF LIMITS for new files
- `context/` is READ-ONLY
- Credentials masked as `[REDACTED]`
- No PII or raw data in logs
- No data exfiltration via web fetches without explicit request
- Each subagent has explicit `tools:` allowlist
- `PREFERENCES.md` cannot override security rules

## Skills

| Skill | What it knows |
|---|---|
| **python** | pandas/NumPy, scikit-learn, visualization, testing, deployment |
| **bigquery** | Partitioning, clustering, joins, HLL++, cost control, bq CLI |
| **excel** | Formulas, dynamic arrays, Power Query M, Power Pivot/DAX, VBA |

Drop your own skill folders into `~/.gemini/skills/`. Each skill is a `SKILL.md` plus optional `references/`.

## Platform

Built for Windows (PowerShell). Works on macOS/Linux — adjust install commands as shown above.

## License

[MIT](LICENSE)
