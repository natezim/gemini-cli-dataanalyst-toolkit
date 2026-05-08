# Detailed Rules

Core rules (file discipline, save-as-you-go, audit log, alignment) are in GEMINI.md.
Session-end logic (versioning, cleanup, chat log) is embedded in `/session:save`.
This file is the always-loaded detail layer.

## Audit log — action codes

Format: `[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>`

Action codes:
- `WRITE` — created a new file
- `EDIT` — modified an existing file
- `DELETE` — removed a file
- `EXEC` — ran a script or shell command
- `QUERY` — ran a database query
- `READ-EXT` — read content from outside the project (web fetch, etc.)
- `VERSION` — snapshotted a file as v1/v2/v3
- `SESSION-START` / `SESSION-END` — session boundaries

Result: `OK`, `FAIL`, or short context (e.g., `142 rows`, `exit 0`, `2.1MB scanned`).
Never log file contents, query data, or credentials — structure only.

## Safety — action classification

Read-only by default. Classify every action:

  SAFE: read operations, dry runs, previews → proceed
  MEDIUM: creating non-production objects, writing to staging → confirm once
  CRITICAL: production modification, permission changes, DDL → explicit approval

- Never modify production unless the user spells out exactly what to change.
- "Clean up the database" is not approval to DROP anything.
- Validate or dry-run before expensive operations when supported.
- Show the full command, report estimated impact, wait for confirmation.

## Query lifecycle

Queries live in `./output/queries/` as `.sql` files. One per query, iterated in place.
Include a comment header: purpose, target, date last modified.

Steps:
  1. Write it.
  2. Validate (syntax check, dry run if available).
  3. Test it. If wrong, fix and go back to 2.
  4. Show the user. Ask: "Does this look right?"
  5. If no: adjust, back to 2. If yes: save and log.

Hygiene:
- No SELECT * — specify columns.
- LIMIT on exploratory queries.
- Always parameterized — never concatenate user input into SQL strings.
- Use cost guardrails when the database supports them.

## Error handling

Show the error, explain in plain language, propose a fix, wait for approval.
Never silently retry. If retrying, vary your approach.
After two failed attempts, stop and ask.

## Anti-patterns

- Do not hard-code environment-specific values.
- Do not produce unrequested output.
- Do not fabricate file contents, API responses, or schema details.
- Do not assume correct retrieval means correct reasoning — verify the logic too.

## Context & memory

- `./context/` is READ-ONLY. Never modify originals.
- `/memory add` persists facts across sessions.
- `/restore` reverts file changes (checkpointing enabled).
- Quality degrades as context grows. Compact or start fresh for new phases.
- For multi-phase work: write findings to files between phases.

## Logging discipline

ONE log: `./output/audit-log.md`. Silent. Only on state changes. See GEMINI.md for triggers.
Format: `[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>`

Use `/memory add` for findings, decisions, learnings — not files.
The chat log at `/session:save` captures user prompts (not Gemini's responses).
