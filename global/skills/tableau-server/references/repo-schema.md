# Repository Schema

## Identity & access
- `system_users`: cross-site login (id, name, email, admin_level, domain_id, hashed_password)
- `users`: site membership (id, site_id, system_user_id, site_role_id, login_at, luid)
- `groups` / `group_users`: site-scoped groups + membership

## Content inventory
- `sites`: isolation boundary; most content keyed by site_id
- `projects`: id, name, site_id, owner_id, parent_project_id, controlled_permissions_enabled
- `workbooks`: id, name, project_id, owner_id, site_id, first/last_published_at, is_deleted (Recycle Bin)
- `views`: id, workbook_id, owner_id, sheettype, site_id, is_deleted
- `datasources`: id, name, repository_url, owner_id, project_id, site_id, size, db_class, db_name

## Connections & origins
- `data_connections`: dbclass, server, port, dbname, tablename, owner_type ("Datasource"/"Workbook"), owner_id, datasource_id (FK when workbook connects to published DS)
  - Custom SQL gotcha: `tablename` is null/generic for Custom SQL — repository cannot parse internal SELECTs. Use Metadata API (GraphQL) for Custom SQL lineage.
  - `db_subclass` column exists in some versions for sub-type tracking
- `extracts`: workbook_id, datasource_id, descriptor, created_at, updated_at
- `api_tokens`: auditing PATs, refresh tokens, OAuth tokens; links via user_id FK to users

## Usage & performance
- `_views_stats` / `views_stats`: user_id, view_id, nviews (cumulative per user-view), time (most recent)
- `_datasources_stats`: datasource_id, nviews, last_access_time
- `_http_requests`: controller, action, uri, status, created_at, completed_at, remote_ip, user_id
  - The underscore view retains ~7d (cleared by `tsm maintenance cleanup`)
  - The raw `http_requests` table retains data per `audit_history_expiration_days` (default 183d)
  - Notable internal actions: render-tooltip-server, set-parameter-value, categorical-filter-by-index
- `_datasources` view: has `id`, `vizportalUrlId`, `luid` — use `vizportalUrlId` or `luid` for browser URLs, NOT `id`

## Jobs & scheduling
- `tasks`: schedule_id, type, priority, obj_type ("Workbook"/"Datasource"), obj_id, consecutive_failure_count
- `schedules`: name, active, schedule_type, run_next_at
- `background_jobs`: created_at, started_at, completed_at, finish_code, job_name, job_type, task_id (NOT FK), correlation_id
- `_background_tasks`: view combining background_jobs + async_jobs (~30d retention)

## Auditing & history
- `historical_events`: historical_event_type_id, worker, is_failure, details, created_at, hist_*_id refs
- `historical_event_types`: type_id, action_type, name
- `hist_*` tables: hist_users, hist_workbooks, hist_views, hist_datasources, hist_data_connections (state at event time)
- Retention: `wgserver.audit_history_expiration_days` (default 183)

## Permissions
- `next_gen_permissions`: authorizable_type/id, grantee_type/id, capability_id, permission (1=grant group, 2=deny group, 3=grant user, 4=deny user)
- `capabilities`: id, name, display_name
- Note: Tableau warns this "does not contain all relevant information" (implicit owner permissions)

## Additional tables
- `datasource_metrics_aggregations`: partial aggregations of datasource analytics per time interval; requires summing across multiple rows for totals
- `datasource_events`: continuous event stream for datasource modification/access
- `resource_holder_types` / `resource_types`: entity and resource type definitions

## Schema versioning
- `schema_migrations`: tracks applied migrations
- Detection: query schema_migrations + information_schema.columns fingerprint on startup
- Maintain template registry per schema family: underscore views (stable) first, deep tables (version-dependent) second

## Built-in user access scopes
- `tableau` user: restricted to tables starting with `_` (underscore views) and `hist_` prefix
- `readonly` user: wider access including granular historical audit and relational mapping tables

## External repository deployment
Repository can be externalized to: Amazon RDS, Google Cloud SQL for PostgreSQL, Azure Database, or standalone PostgreSQL.

## Common join keys
- `site_id`: pervasive partition key
- `owner_id` → `users.id`
- `project_id` → `projects.id`
- `workbook_id` → `workbooks.id`
- `system_user_id` → `system_users.id`
- `correlation_id`: ties subscriptions to background jobs
