# Security, Operations & Dashboard Design

## Enabling repository access
```bash
tsm data-access repository-access enable \
  --repository-username readonly \
  --repository-password <PASSWORD>
```
Opens port 8060. **Restarts Tableau Server.** Always warn before suggesting.

## Security controls (mandatory)
- **Forbid writes**: DB role = SELECT only. SQL template allowlists. Reject DDL/DML.
- **Least privilege**: prefer `tableau` user (stable views); `readonly` only with whitelisted templates
- **PII masking**: `_http_requests` has client IPs — mask by default
- **TLS**: Postgres SSL, RMT HTTPS, secure JMX (SSL + mTLS)
- **Secrets**: store passwords/PATs in secrets manager (Vault, Google Secret Manager)
- **Parameterized SQL only**: bind parameters, never string-interpolate

## Retention windows
| Source | Retention |
|---|---|
| `historical_events` | 183 days (configurable: `wgserver.audit_history_expiration_days`) |
| `_http_requests` | ~7 days (cleanup: `tsm maintenance cleanup --http-requests-table`) |
| `_background_tasks` | ~30 days (auto-cleaned) |

Dashboards exceeding these must: (a) label coverage window, or (b) warehouse data.

## Backup & restore
- `tsm maintenance backup` → `.tsbak` (repository + File Store)
- `tsm maintenance restore` → restores from .tsbak; high-impact, gate with confirmation
- Historical events collection impacts backup size

## HA & failover
- `tsm topology failover-repository` for manual failback/failover
- "Preferred active repository" node influences which instance is active on startup
- Connection logic must tolerate host changes — prefer service discovery over hardcoded hostname

## Warehouse-first strategy
1. Extract incremental, time-bounded metrics from repository + RMT
2. Persist to warehouse (Postgres/BigQuery/etc.)
3. Build dashboards against warehouse for long-range trends + reduced operational load

### Airflow example (hourly extraction)
```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

def extract_background_sla(**_):
    repo = PostgresHook(postgres_conn_id="tableau_repo_readonly")
    end_ts = datetime.now(timezone.utc)
    start_ts = end_ts - timedelta(hours=1)
    sql = """
      SELECT job_name, job_type, finish_code, COUNT(*) AS jobs,
        AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_s
      FROM _background_tasks
      WHERE started_at >= %s AND started_at < %s AND completed_at IS NOT NULL
      GROUP BY job_name, job_type, finish_code
    """
    df = repo.get_pandas_df(sql, parameters=(start_ts, end_ts))
    # persist to warehouse
```

## Dashboard KPIs (recommended)
- **Infrastructure (RMT)**: CPU, memory, disk, process status, incidents
- **Interactive performance**: request latency P50/P95, top slow endpoints
- **Background operations**: extract refresh runtime/failures, subscription failures, backlog
- **Usage**: top views, active users, dormant content
- **Governance**: connection origins, permission snapshots, audit events

## Integration methods (use the right one)

| Method | Use for | Risk |
|---|---|---|
| Repository underscore views | Admin KPIs, stable semantics | Retention limits, operational load |
| Repository deep tables | Detailed analytics (version-dependent) | Schema drift |
| REST API / TSC | State changes, content management | Token handling |
| Metadata API (GraphQL) | Lineage, impact analysis | Disabled by default; enable via TSM |
| RMT repository | Infra/perf monitoring | Requires Advanced Management |
| JMX | JVM process metrics | Security-sensitive, admin-enabled |
| Hyper API | Extract file automation | Not for ops telemetry |

## Sensitive metadata exposure risk
Compromised Workgroup database reveals: confidential dashboard titles, executive user
activity patterns, exact IP addresses and database names of upstream infrastructure.
Use RLS with `USERNAME()` on admin dashboards to restrict by viewer identity.

## Monitoring paradox
Heavy analytical queries against the operational repository can degrade production performance.
ETL pipeline to warehouse solves this: the monitoring doesn't slow down what it monitors.

## Tableau Server Insights (community resource)
Community-maintained GitHub repo with curated `.tds` files, version-controlled per Tableau Server release:
- **TS Events**: stitches historical_events + types + hist_* tables; resolves complex FKs
- **TS Users**: links site users to system users; identifies Creator-licensed users with Viewer behavior
- **TS Data Connections**: handles polymorphic owner_type/owner_id joins
- **TS Background Tasks**: join logic linking projects, system_users, licensing_roles; avoids parsing `args`
