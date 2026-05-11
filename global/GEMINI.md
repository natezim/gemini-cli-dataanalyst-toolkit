# GEMINI.md — Global Rules

Lives at `~/.gemini/GEMINI.md`. Auto-loaded by Gemini CLI in every project, every session.

`./CONTEXT.md` and `./MEMORY.md` from the working directory are auto-loaded too (configured in `~/.gemini/settings.json`).

## File discipline (in every project)

| File type | Folder |
|---|---|
| `.sql` (queries) | `./output/queries/` |
| `.py`, `.js`, `.sh`, `.ipynb`, `.yaml`, `.toml` | `./output/code/` |
| `.md` (reports, docs) | `./output/reports/` |
| `.csv`, `.json`, `.xlsx`, `.parquet` | `./output/data/` |
| Throwaway test/debug | `./output/temp/` (auto-wiped at session end) |
| Audit log | `./output/audit-log.md` |
| Handoff docs | `./output/<date>_handoff.md` |

- Project root is OFF LIMITS for new files. The kit's project-template ships with the standard files; nothing else belongs at root.
- `./context/` is READ-ONLY — copy to `./output/code/` to work with it.
- ONE file per deliverable. Iterate in place. No `_v2`, `_final`, `_new` suffixes — versions are auto-managed at `/session:save`.
- For one-off shell checks: run inline, no file.
- **THROWAWAY = `./output/temp/`, NO EXCEPTIONS.** Test scripts, scratch SQL, debug helpers, profiling probes — all go in `temp/`. It auto-wipes at session end. Never put exploratory work in `code/`, `reports/`, `queries/`, or `data/` — those are reserved for deliverables.

## Audit log (silent, state changes only)

Append ONE line to `./output/audit-log.md` when you:
  → WRITE / EDIT / DELETE a file in queries/, code/, reports/, or data/
  → EXEC a script or non-trivial shell command
  → QUERY a database that returned data
  → READ-EXT (web fetch, external content)

Format: `[YYYY-MM-DD HH:MM:SS] | <user> | <ACTION> | <target> | <result>`

Action codes:
- `WRITE` created a file, `EDIT` modified, `DELETE` removed
- `EXEC` ran a script or non-trivial shell command
- `QUERY` ran a database query (result: row count or scan size)
- `READ-EXT` read external content (web fetch)
- `VERSION` snapshotted a file as v1/v2/v3 at session save

Result: `OK`, `FAIL`, or short context (`142 rows`, `exit 0`, `2.1MB scanned`).

DO NOT log: file reads, list_directory, grep_search, timestamp commands, tool calls that didn't change state, asking the user a question, session boundaries.

Bulk operations = ONE summary entry, not N entries.

LOGGING IS SILENT. Never narrate, never ask permission, never mention it. If user asks what was logged, then show them.

Never log file contents, query data, or credentials — structure only.

## Memory — three tiers

1. `~/.gemini/GEMINI.md` — this file. Global rules (you're reading them).
2. `./CONTEXT.md` — project description, conventions, schemas. User-written.
3. `./MEMORY.md` — project-private dynamic state. **Agent-managed.** Two sections:
   - `## Active threads` — what's open right now
   - `## Learned` — durable facts about this project

The user's slash command `/memory add` is for THEM to type (writes to `~/.gemini/GEMINI.md`); you don't invoke it. You write to `./MEMORY.md` as a regular file.

### How you maintain MEMORY.md

**Active threads** (current state — append/update freely):
```
- **<thread name>** — last touched YYYY-MM-DD
  - Why: <one line on what this matters for>
  - Status: <current state>
  - Files: <key files>
  - Next: <concrete next action>
```
Order: "Why" first. Mark `[STALE >30d]` if last-touched is more than 30 days ago.

**Learned** (durable facts — gated by recurrence):
```
- [YYYY-MM-DD] <fact> — source: <session or file ref>
```
**Do NOT append on first observation.** Stage candidates in the handoff doc's `## Proposed learnings` section. They get promoted to MEMORY.md only after they survive into a second session (either re-observed or user-confirmed via /start). This is the recurrence gate — prevents fabricated "learnings" from polluting permanent memory.

What belongs in Learned:
- Project-specific quirks ("orders.created_at is UTC; reports use ET")
- Recurring user preferences ("user always wants p99 not p95")
- Schema/system gotchas discovered the hard way

What does NOT belong:
- One-off facts that won't matter next session
- Anything derivable from `./context/`
- Single-observation speculation

**Pruning by size, not just age.** If MEMORY.md grows past ~100 lines at `/session:save`, archive (silently delete) the oldest entries that are: stale Active threads >60 days, or Learned facts >180 days that haven't been referenced.

## Core behavior

- Be concise. No preamble. Don't ask "does that meet expectations" — let the user volunteer.
- Prefer the simplest solution.
- If you cannot access a file, say so. NEVER fabricate contents.
- Verify with execution / dry runs — not self-reflection.
- If ambiguous: ask. If corrected: acknowledge and adjust.

## Platform — Windows

Do NOT use Unix commands (grep, cat, ls, cp, mv, rm, find, head, tail) — they fail in cmd/PowerShell.
Use Gemini built-ins: read_file, list_directory, glob, grep_search.
Shell when needed: PowerShell (Select-String, Get-Content, Get-ChildItem, Copy-Item).

## Execution gates (classification)

- **SAFE / LOW** — reads, dry runs, previews → proceed.
- **MEDIUM** — non-production writes, staging changes → confirm once.
- **HIGH** — production-adjacent multi-step work → show full plan in text, require explicit yes before any writes.
- **CRITICAL** — production modification, DDL, permission changes → hard stop, multi-step approval, never proceed unilaterally.

"Clean up the database" is NOT approval to DROP anything. Always show full command + estimated impact + wait for explicit yes on HIGH/CRITICAL.

## Query lifecycle

Queries live in `./output/queries/` as `.sql` files. One per query, iterated in place. Header comment: purpose, target, date last modified.

1. Write it.
2. Validate (syntax check, dry run if available).
3. Test. If wrong, fix and back to 2.
4. Save and log. For HIGH/CRITICAL (production, DDL, large cost), show the user and wait for yes before running.

Hygiene: no `SELECT *`, `LIMIT` on exploratory queries, always parameterized, cost guardrails when DB supports them.

## Error handling

Show the error, explain in plain language, propose a fix, wait for approval.
Never silently retry. If retrying, vary the approach.
After two failed attempts, stop and ask.

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
- `/restore` reverts any file change (built-in, checkpointing on).

## Skills

When the user's task matches a skill in `~/.gemini/skills/`, activate it automatically. Load multiple if relevant.

Plan Mode gates skill activation — if Plan Mode is on (default), confirm with the user before letting a skill auto-activate during planning research. This is a Gemini built-in safety; nothing to configure beyond `general.plan.enabled: true`.

## Subagents

Specialized subagents live in `~/.gemini/agents/`. Run in isolated context windows.

- Invoke explicitly with `@<name>` — more reliable than implicit routing.
- Built-ins: `@generalist`, `@codebase_investigator`, `@cli_help`.
- Read-only agents (validators, explorers, profilers, reviewers, auditors) can run in parallel.
- State-mutating agents (writers, refactorers) MUST run sequentially. Never dispatch two writers at the same target.
- Each subagent returns a compressed summary, not raw tool output. Trust the summary; don't re-run the work.

## Anti-patterns

- No hard-coded environment-specific values.
- No unrequested output.
- No fabricated file contents, API responses, or schema details.
- Correct retrieval ≠ correct reasoning. Verify the logic too.

## Task completion

Not complete until:
1. Deliverable written to the correct output folder.
2. Output tested.
3. Audit log entry appended (if state changed).
4. MEMORY.md `## Active threads` reflects the new state.

## Session

`/start` to begin (handles first-time setup and resume). `/session:save` to end (versioning, MEMORY.md updates, handoff diff).
