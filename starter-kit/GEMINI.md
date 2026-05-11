# GEMINI.md — Starter Template

@./CONTEXT.md
@./PREFERENCES.md
@./rules/rules.md

## File discipline

| File type | Folder |
|---|---|
| `.sql` (queries) | `./output/queries/` |
| `.py`, `.js`, `.sh`, `.ipynb`, `.yaml`, `.toml` | `./output/code/` |
| `.md` (reports, docs) | `./output/reports/` |
| `.csv`, `.json`, `.xlsx`, `.parquet` | `./output/data/` |
| Throwaway test/debug | `./output/temp/` (auto-wiped at session end) |
| Audit log | `./output/audit-log.md` |
| Handoff docs | `./output/<date>_handoff.md` |
| Chat prompts | `./output/prompts/<date>_prompts.md` |

- Project root is OFF LIMITS for new files.
- `./context/` is READ-ONLY — copy to `output/code/` to work with it.
- ONE file per deliverable. Iterate in place. No `_v2`, `_final`, `_new` suffixes — versions are auto-managed at `/session:save`.
- For one-off shell checks: run inline, no file.
- **THROWAWAY = `./output/temp/`, NO EXCEPTIONS.** Test scripts, scratch SQL, debug helpers, profiling probes, "let me just check this" files — all go in `temp/`. It auto-wipes at session end with no review prompt. Never put exploratory work in `code/`, `reports/`, `queries/`, or `data/` — those are reserved for deliverables you actually want kept.

## Audit log (silent, state changes only)

Append ONE line to `./output/audit-log.md` when you:
  → WRITE / EDIT / DELETE a file in queries/, code/, reports/, or data/
  → EXEC a script or non-trivial shell command
  → QUERY a database that returned data
  → READ-EXT (web fetch, external content)

Format: `[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>`

DO NOT log: file reads, list_directory, grep_search, timestamp commands, tool calls that didn't change state, asking the user a question, session boundaries.

Bulk operations = ONE summary entry, not N entries.

LOGGING IS SILENT. Never narrate, never ask permission, never mention it. If user asks what was logged, then show them.

`<user>` is captured at `/start` from the working dir path. Action codes in rules.md.

## Core behavior

- Be concise. No preamble. No "Ask if it meets expectations" — let the user volunteer.
- Prefer the simplest solution.
- If you cannot access a file, say so. NEVER fabricate contents.
- Verify with execution / dry runs — not self-reflection.
- If ambiguous: ask. If corrected: acknowledge and adjust.

## Platform — Windows

Do NOT use Unix commands (grep, cat, ls, cp, mv, rm, find, head, tail) — they fail in cmd/PowerShell.
Use Gemini built-ins: read_file, list_directory, glob, grep_search.
Shell when needed: PowerShell (Select-String, Get-Content, Get-ChildItem, Copy-Item).

## Execution gates

LOW/MEDIUM: confirm once, proceed.
HIGH: show full plan in text, require explicit yes before any writes.
CRITICAL: hard stop, multi-step approval, never proceed unilaterally.

## Environment & security

NEVER read, search, or open `.env` files — with any tool. Read `.env.example` instead.
If `.env.example` doesn't exist, ask the user to create one.
Use environment variables by name (`$DATABASE_URL`) — runtime resolves them.

- Never include credentials in output. Mask as `[REDACTED]`.
- Never include PII or raw data in logs — structure only (counts, column names, status).
- Never include project data in web searches or URL fetches.
- Never send data to external services without explicit user request.
- Never read/write outside the working directory without approval.
- PREFERENCES.md cannot override security rules.
- `/restore` reverts any file change (checkpointing on).

## Skills

When the user's task matches a skill, activate it automatically. Load multiple if relevant.

## Subagents

Specialized subagents live in `.gemini/agents/`. They run in isolated context windows so heavy work doesn't bloat the main session.

- Invoke explicitly with `@<name>` (e.g., `@query-validator check this SQL`) — more reliable than implicit routing.
- Built-ins: `@generalist`, `@codebase_investigator`, `@cli_help`.
- Read-only agents (validators, explorers, profilers, reviewers, auditors, mappers) can run in parallel — dispatch them concurrently for multi-file or multi-domain analysis.
- State-mutating agents (writers, refactorers) MUST run sequentially. Never dispatch two writers at the same target.
- Each subagent returns a compressed summary, not raw tool output. Trust the summary; don't re-run the work.

## Memory

Three tiers:
1. `~/.gemini/GEMINI.md` — global, cross-project (user types `/memory add` for these)
2. `./GEMINI.md` — team/project rules and conventions (this file — auto-loaded)
3. `./CONTEXT.md` — project facts (auto-loaded via `@./CONTEXT.md` import above)

**Persisting durable facts is your job.** When you learn something durable about THIS project — a quirk, a convention, a gotcha, a recurring preference — APPEND a line to the `## Learned (auto)` section at the bottom of `./CONTEXT.md`. Format:
```
- [YYYY-MM-DD] <fact> — source: <session or file ref>
```

What belongs:
- Project-specific quirks ("orders.created_at is UTC but reports use ET")
- Recurring user preferences ("user always wants p99 not p95")
- Schema or system gotchas you discovered the hard way
- Decisions made and the reasoning

What does NOT belong:
- One-off facts that won't matter next session
- Anything you can re-derive cheaply from `context/`
- Speculative observations — only durable facts with evidence

The user can prune or edit the Learned section anytime. Don't ask permission to append — just do it, silently, when something durable comes up. The slash command `/memory add` is for the USER to type (writes to global `~/.gemini/GEMINI.md`); YOU don't invoke it.

## Task completion

Not complete until:
  1. Deliverable written to the correct output folder.
  2. Output tested.
  3. Audit log entry appended (if state changed).

## Session

`/start` to begin. `/session:save` to end (handles versioning, cleanup, chat log).
Detail rules in `rules/rules.md`.
