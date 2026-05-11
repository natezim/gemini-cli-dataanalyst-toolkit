# Gemini CLI — Data Analyst Toolkit Guide

A Gemini CLI setup for data analysts, engineers, and scientists. **9 specialized subagents**, smart session-resume, audit trail, versioning, and skills for Python / BigQuery / Excel.

## Setup

### 1. Global install (once per machine)

```powershell
Copy-Item -Recurse global\* "$env:USERPROFILE\.gemini\"
```

```bash
cp -r global/* ~/.gemini/
```

Installs `GEMINI.md` (rules), `PREFERENCES.md`, `settings.json`, all 9 custom agents, 3 skills, custom slash commands.

### 2. Per-project setup

```powershell
Copy-Item -Recurse <kit-repo>\project-template\* .
```

Then run `/start`.

### 3. Updates

Pull, copy `global/*` to `~/.gemini/` again. All projects updated.

## How it works

### `/start` — start or resume

Identifies you from the working directory, snapshots files, and runs first-time setup OR resumes.

**Returning session:** reads the most recent `output/<date>_handoff.md`. Promotes confirmed `## Proposed learnings` to `MEMORY.md`. Highlights stale `Active threads` (>30 days). Says *"Welcome back. Open threads: X. What are we working on?"*

### Skills

Auto-activate when your task matches.

### Subagents

Type `@` in chat — picker shows 9 custom + 3 built-ins. Each runs in isolated context. See [AGENTS.md](AGENTS.md) for the decision tree.

### Plan Mode

On by default. Pro researches, Flash implements, you approve risky work. Trivial requests skip planning.

### File discipline

| Type | Goes in |
|---|---|
| `.sql` | `output/queries/` |
| `.py`, `.js`, `.sh`, configs | `output/code/` |
| `.md` (reports) | `output/reports/` |
| `.csv`, `.json`, `.xlsx`, `.parquet` | `output/data/` |
| Throwaway | `output/temp/` (auto-wiped at session end) |

### Memory — 3 tiers

| File | Owner | What |
|---|---|---|
| `~/.gemini/GEMINI.md` | You (via `/memory add`) | Global cross-project |
| `<project>/CONTEXT.md` | You | Project description, conventions |
| `<project>/MEMORY.md` | Agent | `## Active threads` + `## Learned` |

Edit `CONTEXT.md` and run `/memory refresh` to apply mid-session.

**Recurrence gate**: agent observes a maybe-fact → stages in handoff → next `/start` promotes if still valid. No fabricated learnings.

### Audit log

Silent. Only state changes. Format:
```
[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>
```

Reads, list_directory, session boundaries NOT logged.

### Versioning

At `/session:save`, modified files snapshot as `_v1`, `_v2`, `_v3` (rotating).

### Handoff doc

Per-session diff:
- What changed
- What failed (verbatim)
- Open / unfinished
- Proposed learnings (staged)
- Next session should…

## Commands

| Command | What it does |
|---|---|
| `/start` | Start or resume |
| `/session:save` | End session — versioning, MEMORY update, handoff |
| `/version <file>` | Manual snapshot to v1/v2/v3 |
| `/info` | Show available commands, agents, layout |
| `/memory add <fact>` | (built-in) Global fact → `~/.gemini/GEMINI.md` |
| `/memory refresh` | (built-in) Re-load files after editing |
| `/restore` | (built-in) Undo a file change |
| `@<agent>` | Invoke a specialized subagent |
| `@./file` | Inject any file mid-session |

## Subagents — Quick Reference

See [AGENTS.md](AGENTS.md).

**Thinking:** `@solution-designer`
**SQL / data:** `@query-validator`, `@schema-explorer`, `@data-profiler`
**Stats / migration:** `@stats-advisor`, `@sas-migrator`
**Excel / Power Query:** `@excel-reviewer`, `@powerquery-reviewer`
**Output:** `@report-drafter`

## Tips

- **`context/` is your library.** Drop schemas, docs.
- **`output/queries/` is your SQL library.** One file per query.
- **`@solution-designer` first** for vague problems.
- **Validate before billable executions.** `@query-validator` does dry-run cost checks.
- **Don't fight versioning.** Iterate in place.

## Sources

- Official Gemini CLI docs (GEMINI.md hierarchy, `/memory`, checkpointing, skills, subagents)
- OWASP AI Agent Security Cheat Sheet
- Google Cloud Community PRAR workflow
- HumanLayer CLAUDE.md research
- AGENTS.md standard
- LLM limitations research (fabrication prevention, recurrence gating)
- Deep research on Gemini subagent architecture (May 2026)
