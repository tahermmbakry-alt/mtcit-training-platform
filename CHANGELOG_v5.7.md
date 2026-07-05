# Savepoint: v5.7 (sync-fix baseline)

This savepoint captures the working, fixed state of the Workshops Evaluation system
after resolving the cross-browser / Google Sheet sync issues. Use this as the
reference point to roll back to if a future change breaks sync again.

## Files in this savepoint
- `Workshops_evaluation_V5_7.html` — client dashboard/app (Arabic RTL, trainee + admin views)
- `Google_Apps_Script_Workshops_Evaluation_V5_7.txt` — Apps Script backend (deploy as Web App)

## Fixes included in this savepoint

### 1. Records sheet no longer mass-overwritten (backend)
`setDatabase()` used to call `materializeDBRecords(db)`, which cleared the entire
**Records** sheet and rewrote it from whatever a single browser's local cache
contained. This caused trainee data to differ between browsers and to vanish
entirely for new/incognito browsers whenever any admin action triggered a sync.
**Fix:** Records are now written *only* incrementally via `appendEvents()` /
`upsertRecord()`. `setDatabase()` never touches the Records sheet.

### 2. Sessions/QuestionBanks no longer mass-overwritten (backend)
`materializeMasterData()` had the same flaw for the **Sessions** and
**QuestionBanks** sheets — clearing and rewriting them from a single browser's
local snapshot, which looked like the sheet "deleting new data every 2–3 seconds"
whenever any admin browser synced.
**Fix:** Sessions and banks are now merged by key (newer `updatedAt` wins per
code/id) against what's already on the sheets, instead of being replaced.

### 3. Read path no longer sources from a stale cache (backend)
`getDatabase()` read `Settings!DB_JSON` first and only fell back to the sheets if
that was empty. After fix #2, the sheets were correctly merged but the Settings
cache could still hold an older, unmerged snapshot — so new/incognito browsers
would load stale/incomplete data from Settings even though the sheets themselves
were correct.
**Fix:** `getDatabase()` now reads sessions/banks directly from the sheets
(`readMasterFromSheets()`) every time. `Settings!DB_JSON` is kept in sync purely
as a mirror/debug artifact and is no longer used for reads.

### 4. Deletions could be silently undone by a lagging background sync (client)
Deleting a trainee record queues an async event; a periodic background sync (every
~60s) merges local + remote data purely by timestamp, with no concept of "this was
just deleted." If that sync landed before the server processed the delete, it could
merge the still-active copy back in.
**Fix:** Added local deletion tombstones (`code/traineeKey → timestamp`) that
`mergeDB()` checks on every merge, so a deleted record can never be resurrected by
a sync unless a genuinely newer update supersedes the deletion. `renderTable()` also
now defensively filters out any record flagged `.deleted` before drawing rows.

### 5. Dead duplicate code removed (client)
An old, unreachable duplicate of `deleteEvaluationRecord()` (overridden by a later
patch layer, but still present in the file) was removed to avoid future confusion
about which implementation actually runs.

## Deployment notes for this savepoint
- Client file: upload/host `Workshops_evaluation_V5_7.html` wherever it's currently served.
- Backend file: paste `Google_Apps_Script_Workshops_Evaluation_V5_7.txt` into the
  Apps Script editor, then **Deploy → Manage deployments → Edit → Version: New
  version → Deploy**. A plain Save does not update the live `/exec` URL.
- No spreadsheet data migration is required — existing Sessions/QuestionBanks/Records
  data is preserved as-is; only the sync behavior going forward changes.

### 6. Admin-only sync buttons hidden from the trainee view (client)
"مزامنة كاملة" / "تحديث من السحابة" / "تهيئة مركزية" are manual admin sync
overrides and were visible (or could briefly flash visible) on the trainee login
screen, which could confuse trainees.
**Fix:** These buttons now default to `display:none` directly in the markup (so
they can never flash visible before JS runs), and `activateRole()` explicitly
reveals them only for admin-facing views (`stats`, `adm`, `qb`, `sec`), hiding
them again for the trainee view. This is purely a visibility change — it does
not alter `manualFullSync()`, `forceCloudRefresh()`, `publishMasterSnapshot()`,
or any background timers; all automatic syncing behaves exactly as before.

## Known remaining cleanup (not required, optional follow-up)
The HTML file still contains multiple overlapping/legacy sync function layers from
earlier patch iterations (only the last-assigned version of each function actually
runs). They're inert but make the file harder to maintain. A future pass could
consolidate them into a single clean sync module without changing behavior.
