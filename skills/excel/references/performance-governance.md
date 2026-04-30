# Performance & Governance

## Calculation modes
- **Automatic**: default, recalculates on every change
- **Manual**: safety valve for large workbooks — but results appear "stale"
- **Automatic except Tables**: hybrid
- Detect and report calculation mode when troubleshooting performance

## Multi-threaded recalculation (MTR)
Enabled since Excel 2007. Uses multiple threads. Scales with CPU cores.
Not all workbooks benefit — depends on formula dependency structure.

## Volatile functions — performance traps
Recalculate on EVERY change even if precedents unchanged:
- NOW, TODAY, RANDBETWEEN, OFFSET, INDIRECT, INFO, CELL

**Replacements**: OFFSET → INDEX, INDIRECT → CHOOSE.
Minimize volatile functions in large workbooks.

User-defined VBA functions: `Application.Volatile` — use carefully.

## Query folding (Power Query)
Folding pushes transforms to data source. Unfolded = all data pulled locally.
- Filter early, select only needed columns
- Check: View Native Query (grayed out = folding stopped)
- Many custom M steps break folding — warn when this happens

## Workbook protection

**Structure protection**: prevent add/move/delete/rename sheets.
**Worksheet protection**: two-step — unlock editable cells first, then protect.
**File encryption**: "Encrypt with Password" — separate from structure/sheet protection.

These are NOT security mechanisms for sensitive data — they prevent accidental changes.

## Co-authoring & AutoSave
- Requires modern `.xlsx` format + OneDrive/SharePoint storage
- AutoSave: saves every few seconds — open read-only to avoid pushing temp changes
- Version history: view/restore previous versions

## Sensitivity labels
Microsoft Purview labels for classification. Tenant settings control co-authoring
for labeled/encrypted documents. Compatibility varies across app versions.

## Macro security
- Trust Center settings control macro execution
- Trusted locations and trusted documents bypass Protected View
- **Protected View**: read-only for untrusted files (internet, other users' OneDrive)
- WARNING: trusting documents bypasses Protected View entirely

## External data refresh security
- Connection files may contain queries — malicious replacement = data exfiltration
- Credential handling must be governed
- Never auto-refresh arbitrary external connections without provenance checks

## Mandatory guardrails
- Never suggest enabling macros without explaining security implications + requiring confirmation
- Never propose auto-refresh external connections without credential/provenance governance
- Treat VBA execution as HIGH risk
- Provide Office Scripts alternatives before VBA when possible
- Desktop vs web feature differences: always warn when action is platform-limited
