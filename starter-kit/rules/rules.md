# Detailed Rules

GEMINI.md has the core. This file is the always-loaded detail layer.

## Action codes (audit log)

- `WRITE` — created a new file
- `EDIT` — modified an existing file
- `DELETE` — removed a file
- `EXEC` — ran a script or shell command
- `QUERY` — ran a database query
- `READ-EXT` — read content from outside the project (web fetch, etc.)
- `VERSION` — snapshotted a file as v1/v2/v3 at session save

Result field: `OK`, `FAIL`, or short context (`142 rows`, `exit 0`, `2.1MB scanned`).
Never log file contents, query data, or credentials.

## Action classification

Read-only by default. Classify every action:

  SAFE: reads, dry runs, previews → proceed
  MEDIUM: non-production writes, staging → confirm once
  CRITICAL: production modification, DDL, permission changes → explicit approval

- "Clean up the database" is not approval to DROP anything.
- Validate or dry-run before expensive operations.
- For CRITICAL: show the full command, estimated impact, wait for explicit yes.

## Query lifecycle

Queries live in `./output/queries/` as `.sql` files. One per query, iterated in place.
Header comment: purpose, target, date last modified.

Steps:
  1. Write it.
  2. Validate (syntax check, dry run if available).
  3. Test it. If wrong, fix and back to 2.
  4. Save and log. For HIGH/CRITICAL queries (production, DDL, large cost), show the user and wait for yes before running.

Hygiene:
- No `SELECT *` — specify columns.
- `LIMIT` on exploratory queries.
- Always parameterized — never concatenate user input into SQL.
- Use cost guardrails when the database supports them.

## Error handling

Show the error, explain in plain language, propose a fix, wait for approval.
Never silently retry. If retrying, vary the approach.
After two failed attempts, stop and ask.

## Anti-patterns

- No hard-coded environment-specific values.
- No unrequested output.
- No fabricated file contents, API responses, or schema details.
- Correct retrieval ≠ correct reasoning. Verify the logic too.
