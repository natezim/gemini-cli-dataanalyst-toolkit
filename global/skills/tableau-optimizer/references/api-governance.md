# Tableau REST API & Metadata API

## REST API basics
- Current version: 3.28. Use `api/-` for latest.
- Auth: Personal Access Tokens (recommended) or username/password.
- Returns `X-Tableau-Auth` token valid 240 minutes.

## Key endpoints

| Operation | Endpoint |
|---|---|
| List workbooks | `GET /api/3.28/sites/{site-id}/workbooks` |
| Workbook details | `GET /api/3.28/sites/{site-id}/workbooks/{id}` |
| Connections | `GET /api/3.28/sites/{site-id}/workbooks/{id}/connections` |
| Download workbook | `GET /api/3.28/sites/{site-id}/workbooks/{id}/content` |
| Download without extract | Append `?includeExtract=False` |
| View usage stats | `GET /api/3.28/sites/{site-id}/views?includeUsageStatistics=true` |
| Extract refresh tasks | `GET /api/3.28/sites/{site-id}/tasks/extractRefreshes` |

REST API does NOT reveal custom SQL — use Metadata API or parse TWB XML.
REST API does NOT expose "last accessed date" — use Server Repository or Metadata API.

## Python TSC library

```python
import tableauserverclient as TSC

auth = TSC.PersonalAccessTokenAuth('TOKEN-NAME', 'TOKEN-VALUE', site_id='CONTENTURL')
server = TSC.Server('https://SERVER', use_server_version=True)

with server.auth.sign_in(auth):
    # Iterate ALL workbooks (auto-pagination)
    for wb in TSC.Pager(server.workbooks):
        server.workbooks.populate_connections(wb)
        for conn in wb.connections:
            print(conn.connection_type, conn.server_address)

    # Download without extract
    path = server.workbooks.download('workbook-luid', filepath='/tmp', include_extract=False)

    # Metadata API via same session
    result = server.metadata.query('{ workbooks { name } }')
```

TSC.Pager is essential — .get() returns only first 100 items.

## Metadata API (GraphQL)

Endpoint: `https://<server>/api/metadata/graphql`
Same auth token as REST. Max page size: 1,000.
On Server: requires `tsm maintenance metadata-services enable`.

### Custom SQL audit (only reliable server-wide method)
```graphql
query {
  customSQLTables {
    name
    query
    connectionType
    isUnsupportedCustomSql
    downstreamWorkbooks { name owner { username } }
  }
}
```

### Calculated field formulas
```graphql
query {
  workbooks {
    name
    embeddedDatasources {
      fields {
        name
        ...on CalculatedField { formula role dataType }
      }
    }
  }
}
```

### Lineage / impact analysis
```graphql
query {
  databaseTables(filter: {name: "orders"}) {
    columns(filter: {name: "customer_id"}) {
      downstreamFields { name datasource { name } }
      downstreamWorkbooks { name owner { username } }
    }
  }
}
```

### Live vs extract detection
```graphql
query {
  publishedDatasources {
    name
    hasExtracts
    extractLastRefreshTime
  }
}
```

### Certification and data quality
```graphql
query {
  publishedDatasources {
    name
    isCertified
    certifierDisplayName
    labels { value category message }
  }
}
```

Key types: Workbook, PublishedDatasource, EmbeddedDatasource, DatabaseTable,
CustomSQLTable, CalculatedField, Column, ColumnField, DatasourceField.
