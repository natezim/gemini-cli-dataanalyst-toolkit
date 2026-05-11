# Production SQL Templates

Always wrap with safe query pattern:
```sql
BEGIN;
SET LOCAL statement_timeout = '5s';
-- query here with :params and LIMIT
COMMIT;
```

## Content inventory
```sql
SELECT s.name AS site, p.name AS project, w.name AS workbook, v.name AS view, v.sheettype
FROM sites s
JOIN projects p ON p.site_id = s.id
LEFT JOIN workbooks w ON w.project_id = p.id AND w.site_id = s.id
LEFT JOIN views v ON v.workbook_id = w.id AND v.site_id = s.id
WHERE (:site_id IS NULL OR s.id = :site_id)
  AND COALESCE(w.is_deleted, false) = false
  AND COALESCE(v.is_deleted, false) = false
ORDER BY s.name, p.name, w.name LIMIT :limit;
```

## Orphan detection (missing owners)
```sql
SELECT w.id, w.name AS workbook, w.owner_id
FROM workbooks w
LEFT JOIN users u ON u.id = w.owner_id AND u.site_id = w.site_id
WHERE w.site_id = :site_id AND COALESCE(w.is_deleted, false) = false AND u.id IS NULL
LIMIT :limit;
```

## Top views (usage ranking)
```sql
SELECT v.name AS view, w.name AS workbook,
  SUM(vs.nviews) AS total_views, MAX(vs.time) AS last_access
FROM views_stats vs
JOIN views v ON v.id = vs.view_id AND v.site_id = vs.site_id
JOIN workbooks w ON w.id = v.workbook_id AND w.site_id = vs.site_id
WHERE vs.site_id = :site_id
  AND COALESCE(v.is_deleted, false) = false
GROUP BY v.name, w.name
ORDER BY total_views DESC LIMIT :limit;
```

## View load latency (_http_requests, ~7d retention)
```sql
SELECT controller, action, COUNT(*) AS requests,
  AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) AS avg_s,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - created_at))) AS p95_s
FROM _http_requests
WHERE site_id = :site_id AND created_at >= :start AND created_at < :end AND completed_at IS NOT NULL
GROUP BY controller, action ORDER BY requests DESC LIMIT :limit;
```

## Background job SLA (_background_tasks, ~30d retention)
```sql
SELECT job_name, job_type, finish_code, COUNT(*) AS jobs,
  AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_s,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))) AS p95_s
FROM _background_tasks
WHERE started_at >= :start AND started_at < :end AND completed_at IS NOT NULL
GROUP BY job_name, job_type, finish_code ORDER BY p95_s DESC NULLS LAST;
```

## Extract refresh durations & failures
```sql
SELECT bj.job_name, bj.finish_code, bj.started_at, bj.completed_at,
  EXTRACT(EPOCH FROM (bj.completed_at - bj.started_at)) AS runtime_s,
  t.obj_type, w.name AS workbook, d.name AS datasource
FROM background_jobs bj
LEFT JOIN tasks t ON t.id = bj.task_id AND t.site_id = bj.site_id
LEFT JOIN workbooks w ON t.obj_type = 'Workbook' AND w.id = t.obj_id AND w.site_id = t.site_id
LEFT JOIN datasources d ON t.obj_type = 'Datasource' AND d.id = t.obj_id AND d.site_id = t.site_id
WHERE bj.site_id = :site_id AND bj.started_at >= :start AND bj.started_at < :end
ORDER BY bj.started_at DESC LIMIT :limit;
```

## Subscription failures
```sql
SELECT bj.job_name, bj.finish_code, bj.started_at,
  s.subject, s.user_id AS subscriber, s.last_sent
FROM background_jobs bj
LEFT JOIN subscriptions s ON s.id = bj.correlation_id AND s.site_id = bj.site_id
WHERE bj.site_id = :site_id AND bj.started_at >= :start AND bj.finish_code <> 0
ORDER BY bj.started_at DESC LIMIT :limit;
```

## Connection origins (lineage proxy)
```sql
SELECT dc.dbclass, dc.server, dc.port, dc.dbname, dc.tablename, dc.owner_type,
  d.name AS datasource, w.name AS workbook
FROM data_connections dc
LEFT JOIN datasources d ON dc.owner_type = 'Datasource' AND d.id = dc.owner_id AND d.site_id = dc.site_id
LEFT JOIN workbooks w ON dc.owner_type = 'Workbook' AND w.id = dc.owner_id AND w.site_id = dc.site_id
WHERE dc.site_id = :site_id ORDER BY dc.dbclass, dc.server LIMIT :limit;
```
Not full lineage — use Metadata API for authoritative column-level lineage.

## Permission snapshots
```sql
SELECT ngp.authorizable_type, ngp.authorizable_id, ngp.grantee_type,
  c.name AS capability, ngp.permission,
  su.name AS user_login, g.name AS group_name
FROM next_gen_permissions ngp
JOIN capabilities c ON c.id = ngp.capability_id
LEFT JOIN users u ON ngp.grantee_type = 'User' AND u.id = ngp.grantee_id AND u.site_id = :site_id
LEFT JOIN system_users su ON su.id = u.system_user_id
LEFT JOIN groups g ON ngp.grantee_type = 'Group' AND g.id = ngp.grantee_id AND g.site_id = :site_id
WHERE ngp.authorizable_type = :type AND ngp.authorizable_id = :id;
```
Permission values: 1=grant group, 2=deny group, 3=grant user, 4=deny user.

## Historical event audits (default 183d retention)
```sql
SELECT he.created_at, het.action_type, het.name AS event, he.is_failure,
  hu.name AS actor, hw.name AS workbook, hv.name AS view, he.details
FROM historical_events he
JOIN historical_event_types het ON het.type_id = he.historical_event_type_id
LEFT JOIN hist_users hu ON hu.id = he.hist_actor_user_id
LEFT JOIN hist_workbooks hw ON hw.id = he.hist_workbook_id
LEFT JOIN hist_views hv ON hv.id = he.hist_view_id
WHERE he.created_at >= :start AND he.created_at < :end
ORDER BY he.created_at DESC LIMIT :limit;
```
