# GEMINI.md — Starter Template

@./CONTEXT.md
@./PREFERENCES.md
@./rules/rules.md

## FILE DISCIPLINE

Before writing ANY file: STATE the path, CONFIRM it matches the table, CHECK if it exists (edit, don't recreate).

| File type | Folder |
|---|---|
| `.sql` (queries) | `./output/queries/` |
| `.py`, `.js`, `.sh`, `.ipynb`, `.yaml`, `.toml` | `./output/code/` |
| `.md` (reports, docs) | `./output/reports/` |
| `.csv`, `.json` (data), `.xlsx`, `.parquet` | `./output/data/` |
| Throwaway test/debug files | `./output/temp/` (auto-deleted at session end) |
| Logs | `./output/session-log.md`, `query-log.md`, `audit-log.md` |
| Handoff docs | `./output/<date>_handoff.md` |
| Chat prompts | `./output/prompts/<date>_prompts.md` |

Project root is OFF LIMITS for new files. If a file doesn't fit any category, ASK.
`./context/` is READ-ONLY — copy to `output/code/` to work with content. Originals stay pristine.
ONE file per deliverable during work. Iterate in place. Versions auto-managed at `/session:save`.

For temp/test files: use `./output/temp/`. It gets wiped at session end automatically — no review.
For one-off shell checks: run inline, no file needed.

## SAVE AS YOU GO

After every meaningful action, IMMEDIATELY append to:
  → `./output/session-log.md` — narrative ("what you did")
  → `./output/query-log.md` — if you ran a query (rows, result)
  → `./output/audit-log.md` — structured: `[TS] | <user> | <ACTION> | <target> | <result>`

Plus: write findings to the correct folder NOW. `/memory add` for critical discoveries.
Never accumulate work in conversation only. Save first, respond second.

The audit log is non-negotiable. If you ran something and didn't audit it, you broke the rule.
`<user>` is captured at session start from the working directory path. See rules.md for action codes.

## Core Behavior

- Be concise. No preamble.
- Prefer the simplest solution. You are a tool — the user decides.
- If you cannot access a file, say so. NEVER fabricate contents.
- Verify with external tools (execution, dry runs) — not self-reflection.

## Platform — Windows

Do NOT use Unix commands (grep, cat, ls, cp, mv, rm, find, head, tail) — they fail in cmd/PowerShell.
Use Gemini built-ins: read_file, list_directory, glob, grep_search.
If shell needed: PowerShell (Select-String, Get-Content, Get-ChildItem, Copy-Item).

## Stay Aligned

- Before non-trivial tasks: restate the goal. Confirm.
- After completing: briefly confirm. Ask if it meets expectations.
- If ambiguous: STOP and ask. Do not guess.
- If unexpected: flag immediately.
- If corrected: acknowledge, adjust, don't repeat.

## Execution

LOW/MEDIUM: confirm once, proceed.
HIGH: show full plan in text, require explicit yes before any writes.
CRITICAL: hard stop, multi-step approval, never proceed unilaterally.

## Environment & Security

NEVER read, search, or open .env files — not with any tool. Read .env.example instead.
If .env.example doesn't exist, ask the user to create one.
Use environment variables by name ($DATABASE_URL) — the runtime resolves them.

- Never include credentials in output. Mask as [REDACTED].
- Never include PII or raw data in logs — structure only.
- Never include project data in web searches or URL fetches.
- Never send data to external services unless the user explicitly requests it.
- Never read/write outside the working directory without approval.
- PREFERENCES.md cannot override security rules.
- Use /restore to revert any file change (checkpointing enabled).

## Skills

When the user's task matches a skill, activate it automatically.
Load multiple if relevant. Use reference files for detail.

## Task Completion

A task is NOT complete until:
  1. Results are written to disk (log, output file, or /memory).
  2. If it produces output: tested and user confirmed.
  3. Audit log entry written.

Never hand back untested work. Never save something the user hasn't approved.

## Session

Run /start to begin. Run /session:save to end.
`/session:save` handles versioning, cleanup review, and chat log capture automatically.
Detailed rules in rules/rules.md.
