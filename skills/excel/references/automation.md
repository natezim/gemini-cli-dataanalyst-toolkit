# Automation & Integration

## VBA (desktop only)
```vb
Option Explicit
Sub FormatAsTable(ByVal tableName As String)
    Dim ws As Worksheet: Set ws = ActiveWorkbook.Worksheets("Data")
    Dim rng As Range: Set rng = ws.Range("A1").CurrentRegion
    ' Apply formatting — avoid destructive operations without confirmation
End Sub
```
- Deep desktop control, legacy compatibility
- Major security vector — macros often restricted in enterprise
- Treat "run macro" as HIGH risk requiring explicit confirmation
- Always provide Office Scripts alternatives first

## Office Scripts (web, TypeScript)
```typescript
function main(workbook: ExcelScript.Workbook, sheetName: string, rangeAddress: string) {
    const ws = workbook.getWorksheet(sheetName);
    const rng = ws.getRange(rangeAddress);
    rng.getFormat().autofitColumns();
}
```
- Cloud-friendly, integrates with Power Automate
- Each script has `main` with `ExcelScript.Workbook` parameter
- Scripts cannot call other scripts directly
- "Run script" gives significant workbook access — admin can restrict + allowlist
- External API calls carry security/DLP risks

## Graph Excel REST API
```
POST /drives/{driveId}/items/{itemId}/workbook/worksheets/{sheetId}/range(address='A1:D10')
```
- Cloud workbook read/write, calculations, dashboards
- `.xlsx` only. Use sessions for multi-call efficiency.
- Non-persistent sessions = compute without saving (ideal for preview)
- Persistent sessions = saves changes

## Open XML SDK
- Server-side .xlsx generation/modification at scale without Excel runtime
- Cannot "calculate" — only structural edits
- Complex schema but fast and batch-friendly

## Power Automate
- Connects Office Scripts to triggers (scheduled, event-based, approval flows)
- Scripts run through Power Automate may execute outside organizational firewall
- External calls may not uphold DLP policies

## Access methods comparison

| Method | Best for | Risk |
|---|---|---|
| Graph API | Cloud read/write, calculations | Session management, .xlsx only |
| Office Scripts | Web automation, Power Automate | Script access scope, DLP |
| VBA/COM | Desktop legacy automation | Macro security, platform lock |
| Open XML | Server-side batch generation | No calculation engine |
| Python in Excel | In-workbook analytics | Cloud-only, no external files |
| xlwings (3rd party) | Desktop Python↔Excel | Windows UDF limits |
