# ARCHIVED: BLOCKING Files

**Archived:** 2026-04-11  
**Reason:** Contradictions resolved — changes from these files have been merged into their respective primary specifications.

---

## Why These Were Archived

These three files were created to document contradictions and gaps in the primary specs. They served their purpose during the ZERO_GUESSWORK_AUDIT. Their content has now been absorbed into the canonical spec files:

| Archived File | Resolution Target | Status |
|---|---|---|
| `BLOCKING File Watcher API Restructuring FW-1.md` | `File Watcher Manager Specification.md` (M15) | ✅ Merged — spec already uses `vscode.workspace.createFileSystemWatcher`; all chokidar references removed |
| `BLOCKING Project Model Persistence Spec Needed.md` | `Project Model Persistence Specification.md` (M16) | ✅ Merged — full spec now exists at M16 with SQLite schema, APIs, tests |
| `BLOCKING Section Manager Merge Algorithm SM-1.md` | `Section Manager Specification.md` (M22) | ✅ Merged — "Append Below" is now the only documented merge strategy; "User Priority" option removed |

---

## Do Not Use These Files

These files are retained for historical reference only. **The canonical specifications are the primary spec files.** Do not implement from these archived files.
