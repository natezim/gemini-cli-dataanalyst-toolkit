# Gemini CLI — Data Analyst Toolkit Guide

A Gemini CLI setup for data analysts, engineers, and scientists. Audit trail,
versioning, query management, and skills for the tools you actually use —
out of the box, with corporate-grade safety baked in.

## Setup

1. Copy `starter-kit/` contents into your project root.
2. Copy relevant skills from `skills/` into `.gemini/skills/` (Python, BigQuery, Excel).
3. Drop reference files into `context/` (schemas, docs, specs — auto-loaded every session, READ-ONLY).
4. Run `/start`. It handles everything — first-time setup and every session after.
5. Run `/info` anytime to see what's available.

Checkpointing is on. Use `/restore` to undo any file change.

## How It Works

**`/start`** is the only command you need to start. First time, it captures your user identity (from working directory path), scans your project, asks smart questions, and builds CONTEXT.md. Every time after, it loads everything and asks what you're working on. Picks up from your last handoff if there is one.

**Skills activate automatically.** If you ask about BigQuery and the bigquery skill is installed, it loads. Same for Python and Excel. No manual activation.

**Queries are managed.** Write → test → show you → confirm → save. Every execution is logged. Your query library lives in `queries/`.

**Files are organized.** Strict folder structure. No scratch sprawl. `output/code/` for scripts, `output/reports/` for analysis, `output/data/` for exports. Pre-write checkpoint forces Gemini to verify the path before every write.

**Versioning is automatic.** At `/session:save`, modified files get snapshotted as `_v1`, `_v2`, `_v3` (max 3, rotating). Live working file keeps its base name.

**Audit trail is automatic.** Every write, exec, and query gets a structured entry in `output/audit-log.md` with timestamp, user, action, target, and result.

**Chat log captures your prompts** at session end (substantive ones only — skips "yes", "ok", "go ahead"). Saved to `output/prompts/<date>_prompts.md`.

**Cleanup review at session end.** Gemini lists files created this session and asks: keep all, delete all, or review one-by-one. No mid-session interruptions.

**`/session:save`** writes the handoff doc, runs versioning, cleanup, chat log capture, and final audit entry.

## What's in the Kit

```
GEMINI.md              Core rules — auto-loaded
rules/rules.md         Detail layer
CONTEXT.md             Project context — built by /start
PREFERENCES.md         Personal preferences + workflow overrides
.env.example           Connection variable names (Gemini reads this, not .env)
.gemini/settings.json  Checkpointing, compression, session retention
.geminiignore          Keeps secrets and PII out of context
.gitignore             Protects credentials and outputs from git

context/               Reference files — READ-ONLY, auto-loaded
queries/               SQL query library — one .sql per query

.gemini/
  commands/            /start, /setup, /session:save, /context:update,
                       /version, /info
  skills/              Skills that activate automatically

output/
  code/                Scripts (.py, .js, .sh, .ipynb, configs)
  reports/             Analysis, docs (.md)
  data/                Exports (.csv, .json, .xlsx, .parquet)
  audit-log.md         Structured audit trail
  session-log.md       Narrative task log
  query-log.md         Query execution log
  prompts/             Your chat prompts saved per session
```

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — versioning, cleanup, chat log, handoff |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/info` | Show available commands and features |
| `/memory add` | Persist a fact across sessions (survives compression) |
| `/restore` | Undo a file change |
| `@./file` | Inject any file into context mid-session |

## Tips

- **`context/` is your library.** Drop schemas, docs, specs — auto-loaded, READ-ONLY.
- **`queries/` is your query library.** One `.sql` per query, iterated in place, execution logged.
- **`PREFERENCES.md` is your customization file.** Tone, style, habits, plus workflow overrides.
- **`/memory add` for critical facts.** Memory survives compression. Chat history doesn't.
- **`/context:update` before closing.** One minute now saves re-explaining next session.
- **Don't fight versioning.** Iterate in place during work. Versioning happens at `/session:save`.

## Sources

- Official Gemini CLI docs (GEMINI.md, /memory, checkpointing, skills)
- OWASP AI Agent Security Cheat Sheet (risk classification)
- Google Cloud Community PRAR workflow (gated execution)
- HumanLayer CLAUDE.md research (instruction design)
- AGENTS.md standard (cross-tool compatibility)
- LLM limitations research (context management, verification, fabrication prevention)
