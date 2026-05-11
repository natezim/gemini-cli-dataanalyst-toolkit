# Security & Access Control

## IAM principles

- Apply least privilege — no principal should have more permissions than needed.
- Grant access to Google Groups representing teams/departments, not individual users.
- Use layered roles: project-level, dataset-level, and table-level.
- Define PK/FK constraints (NOT ENFORCED) for optimizer benefits, but maintain data integrity yourself.

## Row-level security (RLS)

Row-level access policies act as filters to show/hide rows based on user/group membership.

```sql
CREATE ROW ACCESS POLICY region_filter
ON dataset.sales
GRANT TO ('group:sales-emea@company.com')
FILTER USING (region = 'EMEA');
```

Rules:
- Do NOT grant table write permissions to users who should only see filtered data
  (they could bypass RLS via bq load or Storage Write API).
- Manage `bigquery.filteredDataViewer` role exclusively through row-level access policies.
- RLS coexists with column-level security and dataset/table/project-level controls.

## Column-level security with policy tags

Use policy tags for fine-grained column access control.

```sql
-- Apply via schema definition or ALTER TABLE
-- Then grant Fine-Grained Reader role on the policy tag
```

- Build a hierarchy of data classes mapping to your business taxonomy.
- A single policy tag can apply to multiple columns — manage many columns with few tags.
- At query time, BigQuery checks column access for each tagged column.

## Dynamic data masking

Layers on top of column-level security. Substitutes null, default, or hashed values
for sensitive columns when the user lacks full access.

- Compatible with non-subquery row access policies.
- Applied on top of row-level security.
- Use for PII fields (email, phone, SSN) where some users need masked access.

## VPC Service Controls

- Create a security perimeter around BigQuery resources.
- All traffic encrypted in transit and at rest.
- Use for compliance-sensitive workloads.

## Minimum IAM roles for common patterns

| Use case | Roles needed |
|----------|-------------|
| Run queries | `bigquery.jobUser` + `bigquery.dataViewer` |
| Create tables | `bigquery.dataEditor` |
| Manage datasets | `bigquery.dataOwner` |
| Admin | `bigquery.admin` (use sparingly) |

## Service account keys blocked by default

Organizations created after May 3, 2024 have `iam.disableServiceAccountKeyCreation`
enforced by default. Keys in public repos are automatically disabled since June 2024.
Prefer impersonation or Workload Identity Federation.

## Audit logging

- Enable Cloud Audit Logs for BigQuery to track all user activity.
- Query audit logs via INFORMATION_SCHEMA or Cloud Logging.
- Data access logs capture who queried what data and when.
