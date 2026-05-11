# Gemini CLI — Data Analyst Toolkit Guide

A Gemini CLI setup for data analysts, engineers, and scientists. 9 specialized subagents, audit trail, versioning, query management, and skills for Python / BigQuery / Excel — out of the box, with corporate-grade safety baked in.

## Setup

1. Copy `starter-kit/` contents into your project root.
2. Copy relevant skills from `skills/` into `.gemini/skills/`.
3. Drop reference files into `context/` (schemas, docs, specs — auto-loaded every session, READ-ONLY).
4. Run `/start`.
5. Run `/info` anytime to see commands and agents.

Checkpointing is on. Use `/restore` to undo any file change.

## How It Works

**`/start`** is the entry point. First time, it captures your user identity (from working directory path), scans your project, asks only what it can't infer from files, and builds CONTEXT.md. Every time after, it loads everything and asks what you're working on. Picks up from your last handoff if there is one.

**Skills activate automatically.** If you ask about BigQuery and the bigquery skill is installed, it loads. Same for Python and Excel.

**Subagents activate explicitly.** Type `@` in chat and pick a specialist (`@solution-designer`, `@query-validator`, etc.). Each runs in an isolated context window so heavy work doesn't bloat the main session. See [AGENTS.md](AGENTS.md) for the decision tree.

**Plan Mode is on.** For multi-step or risky work, Gemini researches first (Pro model), proposes a plan, waits for your approval, then executes (Flash model). Trivial requests skip planning automatically.

**Queries are managed.** Write → validate (`@query-validator`) → save → log. Every execution is logged. Your query library lives in `output/queries/`.

**Files are organized.** Strict folder structure. No scratch sprawl. `output/code/` for scripts, `output/reports/` for analysis, `output/data/` for exports. `output/temp/` for throwaway work — auto-wiped at session end.

**Versioning is automatic.** At `/session:save`, modified files get snapshotted as `_v1`, `_v2`, `_v3` (max 3, rotating). Live working file keeps its base name.

**Audit trail is automatic and silent.** Every state-changing action writes one line to `output/audit-log.md` with timestamp, user, action, target, and result. Reads, list_directory, and session boundaries are NOT logged.

**Memory is 3-tier:**
1. `~/.gemini/GEMINI.md` — global cross-project (user types `/memory add`)
2. `./GEMINI.md` — team/project rules
3. `./CONTEXT.md` — project facts you wrote, plus a `## Learned (auto)` section the agent appends to as it discovers durable facts

**Chat log captures your prompts** at session end. Substantive ones only — skips "yes", "ok", etc.

**`/session:save`** writes the handoff doc, runs versioning, wipes `output/temp/`, captures chat log. No prompts about deliverable files — they're kept by rule.

## What's in the Kit

```
GEMINI.md              Core rules — auto-loaded
rules/rules.md         Detail layer
CONTEXT.md             Project context — built by /start
PREFERENCES.md         Personal preferences + agent + workflow overrides
.env.example           Connection variable names (Gemini reads this, not .env)
.gemini/settings.json  Checkpointing, Plan Mode + model routing, compression
.geminiignore          Keeps secrets and PII out of context
.gitignore             Protects credentials and outputs from git

context/               Reference files — READ-ONLY, auto-loaded

.gemini/
  commands/            /start, /setup, /session:save, /context:update,
                       /version, /info
  skills/              Skills that activate automatically
  agents/              9 subagents that activate via @name

output/                Workspace — everything Gemini produces
  queries/             SQL query library — one .sql per query
  code/                Scripts (.py, .js, .sh, .ipynb, configs)
  reports/             Analysis, docs (.md)
  data/                Exports (.csv, .json, .xlsx, .parquet)
  temp/                Throwaway test files — auto-wiped at session end
  audit-log.md         Structured audit trail (silent, only on state changes)
  prompts/             Your chat prompts saved per session
```

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume a session |
| `/setup` | Rebuild context from scratch |
| `/session:save` | End session — versioning, temp wipe, chat log, handoff |
| `/version <file>` | Manual snapshot of a file as v1/v2/v3 |
| `/context:update` | Add something permanent to CONTEXT.md |
| `/info` | Show available commands, agents, and features |
| `/memory add` | Persist a fact across sessions (built-in) |
| `/restore` | Undo a file change (built-in) |
| `@<agent>` | Invoke a specialized subagent |
| `@./file` | Inject any file into context mid-session |

## Subagents — Quick Reference

See [AGENTS.md](AGENTS.md) for the full decision tree with example prompts.

**Thinking:** `@solution-designer` (brainstorm, decompose, weigh tradeoffs)

**SQL / data:** `@query-validator`, `@schema-explorer`, `@data-profiler`

**Stats / migration:** `@stats-advisor`, `@sas-migrator`

**Excel / Power Query:** `@excel-reviewer`, `@powerquery-reviewer`

**Output:** `@report-drafter` (writes to `output/reports/`)

## Tips

- **`context/` is your library.** Drop schemas, docs, specs — auto-loaded, READ-ONLY.
- **`output/queries/` is your query library.** One `.sql` per query, iterated in place, execution logged.
- **`PREFERENCES.md` is your customization file.** Tone, agent defaults, workflow overrides.
- **`/memory add` for critical facts.** Memory survives compression. Chat history doesn't.
- **`/context:update` before closing** when something's worth persisting permanently.
- **Don't fight versioning.** Iterate in place during work. Versioning happens at `/session:save`.
- **Stuck on a vague problem? Try `@solution-designer` first.** It asks the questions that turn "build a dashboard" into a concrete plan.
- **Validate before executing on billable systems.** `@query-validator` does dry-run cost checks before BigQuery scans.

## Sources

- Official Gemini CLI docs (GEMINI.md, /memory, checkpointing, skills, subagents)
- OWASP AI Agent Security Cheat Sheet (risk classification)
- Google Cloud Community PRAR workflow (gated execution)
- HumanLayer CLAUDE.md research (instruction design)
- AGENTS.md standard (cross-tool compatibility)
- LLM limitations research (context management, verification, fabrication prevention)
- Deep research on Gemini subagent architecture (May 2026 release notes)
