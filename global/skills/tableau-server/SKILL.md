---
name: tableau-server
description: >
  Tableau Server administration via the PostgreSQL repository and RMT. Use when the user
  is: monitoring Tableau Server health, querying the repository for usage/content/jobs,
  troubleshooting extract refresh failures, auditing permissions or historical events,
  building admin dashboards, checking background job SLAs, investigating slow views,
  managing subscriptions, tracking connection origins, or working with the Resource
  Monitoring Tool (RMT). Also use for TSM commands, repository access setup, or
  warehouse extraction patterns for Tableau Server metrics.
---

# Tableau Server Admin

Production-grade Tableau Server monitoring and management via the PostgreSQL repository,
RMT, REST API, and Metadata API.

## Critical rules — always apply

### NEVER modify the repository
Tableau explicitly warns: modifications break Tableau Server and void support.
All repository access must be read-only, allowlisted, time-bounded.
Reject any query containing INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE.

### Always parameterize SQL
Use bind parameters (`%s` or named placeholders). Never string-interpolate.
Always include `LIMIT`. Always set `statement_timeout`.

```sql
BEGIN;
SET LOCAL statement_timeout = '5s';
-- SELECT ... with :params and LIMIT
COMMIT;
```

### Know your retention windows
- `_http_requests`: ~7 days (cleanup dependent)
- `_background_tasks`: ~30 days
- `historical_events`: default 183 days (`wgserver.audit_history_expiration_days`)

Never build dashboards exceeding these windows without warehousing data first.
Always label coverage windows on dashboards.

### Layered access — use the right tool
- **REST API / TSC**: state changes (publish, users, groups, jobs)
- **Metadata API (GraphQL)**: lineage and impact analysis (disabled by default)
- **Repository underscore views**: admin reporting (`_background_tasks`, `_http_requests`, `_views_stats`)
- **RMT repository**: infrastructure health, performance metrics, incidents

### Two built-in repository users
- `tableau`: stable views, Tableau limits changes to avoid breaking custom views
- `readonly`: more tables, but internal and may change without warning

### Enabling repository access restarts Tableau Server
`tsm data-access repository-access enable` requires a restart. Always warn before suggesting this.

### PII awareness
`_http_requests` includes client IPs (`remote_ip`, `user_ip`). Mask by default.

---

## Reference files

| File | Load when... |
|------|-------------|
| `references/repo-schema.md` | Repository table/view structure, join keys, column details, schema versioning |
| `references/queries.md` | Production SQL templates (content inventory, usage, jobs, permissions, auditing) |
| `references/rmt.md` | Resource Monitoring Tool architecture, .tds surfaces, access setup, complementarity |
| `references/security-ops.md` | Authentication, backup/restore, HA/failover, warehouse patterns, scheduling, dashboard design |
