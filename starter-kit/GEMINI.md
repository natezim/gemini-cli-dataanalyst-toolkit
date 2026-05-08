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

## Task completion

Not complete until:
  1. Deliverable written to the correct output folder.
  2. Output tested.
  3. Audit log entry appended (if state changed).

## Session

`/start` to begin. `/session:save` to end (handles versioning, cleanup, chat log).
Detail rules in `rules/rules.md`.
