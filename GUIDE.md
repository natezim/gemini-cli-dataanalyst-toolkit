# Gemini CLI — Data Analyst Toolkit Guide

Built for data analysts working with SQL, SAS, Python, BigQuery, Excel, Power Query, Power BI, and Tableau. **13 specialized subagents**, **7 skills**, smart session-resume, query management, production safety — all modular.

## Setup

### 1. Global install (once per machine)

See [README.md](README.md) for the full install menu (everything vs. bundles). Simplest:

```powershell
Copy-Item -Recurse global\* "$env:USERPROFILE\.gemini\"
```

```bash
cp -r global/* ~/.gemini/
```

This installs everything: `GEMINI.md` (rules), `PREFERENCES.md`, `settings.json`, all 13 custom subagents, all 7 skills, custom slash commands.

Pick a bundle (Core + Stats + Power BI etc.) if you don't want the whole set — see README.

### 2. Per-project setup

```powershell
Copy-Item -Recurse <kit-repo>\project-template\* .
```

Then run `/start`. First time, scans the project, asks only what it can't infer, writes `CONTEXT.md`. After that, every `/start` resumes from your last session.

### 3. Updates

Pull, copy the installed parts to `~/.gemini/` again. All projects updated. Project files untouched.

## How it works

### `/start` — start or resume

Captures user identity, snapshots files, runs first-time setup OR resumes.

**Returning session:** reads the most recent `output/<date>_handoff.md`. Applies the recurrence gate: if a proposed learning appears in TWO consecutive handoffs, promotes to `MEMORY.md`. Highlights stale `Active threads` (>30 days).

After a long gap, `/start` says: *"Welcome back. 3 open threads, one stale. 2 facts re-confirmed — promoting. What are we working on?"*

### Skills

Auto-activate when your task matches. Install only what you actually use.

### Subagents

Type `@` in chat — picker shows whichever agents you installed + 3 built-ins. Each runs in an isolated context. See [AGENTS.md](AGENTS.md) for the decision tree.

### Plan Mode

On by default. Pro researches, Flash implements, you approve risky work. Also gates skill activation.

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

**Recurrence gate**: agent stages in handoff → next session re-observes → `/start` promotes to `MEMORY.md`. Two consecutive handoffs = real recurrence. No self-judgment.

### Audit log

Silent. State changes only. Format:
```
[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>
```

### Versioning

At `/session:save`, modified files snapshot as `_v1`, `_v2`, `_v3` (rotating).

### Handoff doc

Per-session diff:
- What changed
- What failed (verbatim)
- Open / unfinished
- Proposed learnings (staged for recurrence gate)
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
**Excel / Power Query / DAX:** `@excel-reviewer`, `@powerquery-reviewer`, `@dax-reviewer`
**BI tools:** `@tableau-auditor`, `@viz-optimizer`, `@migration-mapper`
**Output:** `@report-drafter`

## Tips

- **`context/` is your library.** Drop schemas, docs.
- **`output/queries/` is your SQL library.** One file per query.
- **`@solution-designer` first** for vague problems.
- **Validate before billable executions.** `@query-validator` does dry-run cost checks.
- **Don't fight versioning.** Iterate in place.
- **Install only what you use.** Kit is modular.

## Sources

- Official Gemini CLI docs (GEMINI.md hierarchy, `/memory`, checkpointing, skills, subagents)
- OWASP AI Agent Security Cheat Sheet
- Google Cloud Community PRAR workflow
- HumanLayer CLAUDE.md research
- AGENTS.md standard
- LLM limitations research (fabrication prevention, recurrence gating)
- Deep research on Gemini subagent architecture (May 2026)
