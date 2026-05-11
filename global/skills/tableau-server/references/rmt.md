# Resource Monitoring Tool (RMT)

## What RMT is
Gathers Tableau Server monitoring data. Requires Advanced Management license.
Identifies causes of slow load times, extract failures, critical issues.

Two components:
- Agent on each Tableau Server node (<5% CPU, <200 MB RAM)
- RMT Server (min 8 physical cores/16 vCPUs, 64 GB RAM, 500 GB SSD)

Can monitor multiple clusters from one RMT instance (separate environments).
Prerequisites: Erlang + RabbitMQ (provisioned during install). SCRAM-SHA-256 required.
Minimum Tableau Desktop 2020.4 for connecting to RMT .tds files.

## Architecture
- RabbitMQ message queue collects agent data → RMT Server → RMT PostgreSQL repository
- Since 2022.3: can use external PostgreSQL (AWS RDS) + external RabbitMQ (AWS AMQ)
- RMT accesses Tableau Server Repository directly "for performance reasons"
  → requires `readonly` password; treat RMT as additional consumer of repository access

## RMT repository schema
Internal table names NOT publicly documented. Access via official `.tds` data sources:

| .tds Surface | What it covers |
|---|---|
| Background Tasks | Extract refreshes, subscriptions, flows |
| Data Queries | Queries executed by Tableau Server |
| Gateway Requests | HTTP/VizQL session request data |
| Incidents | Issues detected by RMT |
| Server Performance | Hardware and process metrics (CPU, memory, disk) |
| Tableau Entities | Sites, projects, workbooks, views collected by RMT |

## Enabling read-only access
```bash
rmtadmin data-access ReadOnly
# Restart RMT DB
# Retrieve password: db.readOnlyPassword
```

For 2022.2 and earlier: edit `postgresql.conf` + `pg_hba.conf` (SCRAM-SHA-256), restart DB.
Security-sensitive — opening RMT DB access requires admin action.

## When to use RMT vs Repository

**Use Tableau Server Repository for:**
- Content inventory, ownership, permissions
- Auditing (historical_events)
- Scheduling (tasks, schedules, background_jobs)

**Use RMT for:**
- Hardware/process health (CPU, memory, disk)
- Incident tracking
- Workload bottlenecks
- Request/query performance (Gateway Requests, Data Queries)

RMT's separate database reduces contention on Tableau Server's operational repository.

## RMT Server microservices
- **Director**: coordination, license queries, cluster health, DB schema upgrades. Log: `director\YYYYMMDD-pts.log`
- **Backgrounder**: telemetry processing, cleanup, incident analysis, notifications. Log: `background\YYYYMMDD-pts.log`
- **Web**: web server/UI, agent registration. Log: `web\YYYYMMDD-pts.log`
- **Host**: process watcher, config file change detection. Log: `host\YYYYMMDD.log`
- PostgreSQL logs: `pgsql\*.log` and `*.csv`

## Default hardware thresholds
- CPU warning: >80% for 5 consecutive minutes
- Memory warning: <8 GB available for 10 consecutive minutes
- Disk warning: <10 GB for 10 minutes; critical: <5 GB

## Telemetry blind spots
- `tabprotosrv`, internal `postgres`, and `gateway` processes NOT profiled by agents
- Sum of process CPU won't match host CPU — unaccounted spikes likely `tabprotosrv` during extract refreshes
- External File Store, External Repository, Independent Gateway invisible to agents — supplement with CloudWatch/Azure Monitor

## Externalization
```bash
rmtadmin master-setup --db-config=external --db-server=<rds> --db-database=<name> \
  --db-port=5432 --db-admin-username=postgres --db-admin-password=<pw>
rmtadmin restart --all
```

## Cross-database dashboard patterns
Bridge Workgroup and RMT databases using Tableau Desktop cross-database relationships:
- **Extract Processing + Hardware Saturation**: `_background_tasks` (extract times) + RMT Server Performance (CPU spikes)
- **Toxic extracts**: background tasks with long runtimes + sustained memory = poorly indexed upstream views or complex Custom SQL
- **Content Staleness + Disk Reclaim**: workbook creation dates + extract sizes + last view access + RMT disk alerts

## Tableau Cloud Admin Insights (SaaS alternative)
Cloud admins cannot connect to PostgreSQL or deploy RMT. Instead:
- "Admin Insights" project provisioned natively per Tableau Cloud site
- Data sources: TS Events, TS Users, Groups, Site Content, Viz Load Times, Job Performance, Permissions, Subscriptions, Tokens
- Retention: 90 days default, 365 days with Advanced Management
- Lacks hardware telemetry. Cross-domain analysis requires Tableau Prep flows.
