# Auth and Admin: Service Accounts, OAuth, Tableau Server Config

## Authentication method decision table

| Scenario | Method | Reason |
|----------|--------|--------|
| Interactive dashboards, individual users | OAuth (saved credentials) | Per-user audit trail, IAM-scoped |
| Scheduled extract refreshes | Service Account JSON (embedded) | Non-interactive, no login required |
| Multi-user Tableau Server | Custom OAuth client on Tableau Server | Users manage own tokens |
| High-security / no static keys | OAuth + Workload Identity Federation | WIF eliminates JSON key files |

**Note:** As of April 2026, Tableau does not natively support Workload Identity Federation (WIF).
WIF support requires a custom connector or API-layer proxy.

---

## Service account setup and security

### Minimum required roles

```bash
# Grant only what Tableau needs — never bigquery.admin
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:tableau-reader@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:tableau-reader@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/bigquery.jobUser"
```

`bigquery.dataViewer` — read table data and metadata
`bigquery.jobUser` — run queries (required to execute any SQL)

For cross-project access (Tableau in project A, data in project B):
Grant `bigquery.dataViewer` on project B, `bigquery.jobUser` on project A (billing project).

### Creating and using the JSON key

```bash
# Create key
gcloud iam service-accounts keys create tableau-sa-key.json \
    --iam-account=tableau-reader@PROJECT_ID.iam.gserviceaccount.com

# Rotate keys every 90 days (audit via):
gcloud iam service-accounts keys list \
    --iam-account=tableau-reader@PROJECT_ID.iam.gserviceaccount.com
```

**Security warnings:**
- JSON key files contain private keys in **plaintext** — treat like passwords
- Do not commit to source control
- Rotate every 90 days
- Disable unused keys immediately: `gcloud iam service-accounts keys disable KEY_ID --iam-account=...`

### Using service account in Tableau Desktop

When publishing a workbook: Data Source tab → Edit Connection → Authentication → select
"Service Account" → browse to JSON key file. Check **"Embed password in connection"** to
allow Server to refresh without prompting.

---

## OAuth setup on Tableau Server

Custom OAuth enables per-user saved credentials and web authoring support.

### Configure custom OAuth client

```bash
tsm configuration set -k oauth.google.client_id -v "<CLIENT_ID>"
tsm configuration set -k oauth.google.client_secret -v "<CLIENT_SECRET>"
tsm configuration set -k oauth.google.redirect_uris -v "https://<TABLEAU_SERVER>/auth/add_oauth_token"
tsm pending-changes apply
```

**Google OAuth console settings:**
- Authorized redirect URIs: `https://<your-tableau-server>/auth/add_oauth_token`
- Scopes required: `bigquery.readonly`, `bigquery.jobs.create`, `cloud-platform`

### The 50 token limit issue

Google enforces a **50 saved token limit per user per OAuth client**. If users connect from
multiple Tableau Desktop installations or re-authenticate frequently, tokens accumulate.
Beyond 50, Google silently invalidates the oldest tokens.

**Symptoms:** Users randomly get authentication errors even though credentials appear saved.

**Fix:** Audit and revoke excess tokens at `myaccount.google.com/permissions`.
Encourage users to revoke old Tableau entries periodically.

---

## Credential management on Tableau Server

### Embedding vs. prompting

| Option | Behavior | Use for |
|--------|----------|---------|
| Embed credentials | All users see same data (service account permissions) | Public/shared dashboards |
| Prompt users | Each user authenticates with their own Google account | Row-level security via BigQuery IAM |
| Use server-managed credentials | Users save OAuth token once via My Account | Recommended for user-level access |

### Server-managed OAuth credential storage

Users save credentials once: Tableau Server → My Account → Connected Apps → Add Credentials.
All workbooks published to Server can use stored credentials without re-prompting.

For Server-managed credentials to work: custom OAuth client must be configured on Tableau Server,
and the `oauth.google.client_id` must match what users authorized.

---

## Row-level security via BigQuery

Two approaches:

**BigQuery IAM column-level security:** Grant users access to specific columns via column-level
access policies. Tableau respects column permissions — unauthorized columns return NULL or error.

**BigQuery row-level access policies (2024+):**
```sql
CREATE ROW ACCESS POLICY sales_region_filter
ON `project.dataset.orders`
GRANT TO ("group:europe-sales@company.com")
FILTER USING (region = 'Europe');
```

With per-user OAuth in Tableau, each user's queries run under their own Google identity,
respecting their row access policies automatically.

**Limitation:** Row access policies work only with OAuth authentication (not service accounts
with embedded credentials, which run as the service account identity).

---

## Credential stripping issue on publish (on-prem Tableau Server)

**Symptom:** Credentials are stripped when publishing from Tableau Desktop to Tableau Server.
The workbook appears to publish successfully but connections fail on Server.

**Cause:** Tableau Server's keychain setting for the connector doesn't have "Embed" enabled.

**Fix:** In Tableau Desktop before publishing → Data Source → Edit Connection → Authentication
dropdown → select "Embed password in connection" (not "Prompt user"). This explicitly marks
credentials for embedding in the published workbook.

For OAuth: Server → Site → Connected Apps must have the OAuth client configured, otherwise
Server can't exchange tokens on behalf of the data source.

Reference: https://help.tableau.com/current/server/en-us/protected_auth.htm
