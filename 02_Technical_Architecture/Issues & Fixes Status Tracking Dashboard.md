# 🔄 Issues & Fixes Status Tracking Dashboard

> ## ✅ STATUS: ALL BLOCKING ISSUES RESOLVED — 2026-04-11
>
> This dashboard was the original issue tracker during the spec-review phase. Every BLOCKING and MUST-FIX item listed below has since been resolved as part of the `ZERO_GUESSWORK_AUDIT.md` remediation (Steps 1-11).
>
> **Canonical spec state:**
> - **A1 / Project Model Persistence** — ✅ RESOLVED. Full spec exists at `04_Implementation_Specs_Phase_1.5/Project Model Persistence Specification.md` with **canonical Module ID M16**. The archived BLOCKING doc that referenced M14 lives at `04_Implementation_Specs_Phase_1.5/ARCHIVED_BLOCKING/` and should NOT be implemented against.
> - **FW-1 / File Watcher API** — ✅ RESOLVED. `File Watcher Manager Specification.md` now uses `vscode.workspace.createFileSystemWatcher()` exclusively. Zero chokidar references remain in the canonical spec.
> - **SM-1 / Section Manager Merge** — ✅ RESOLVED. "Append Below" is the only merge strategy (see `Section Manager Specification.md:232-263`). The Build Prompt at the bottom of that file has also been reconciled.
> - **SM-4 / Concurrency tests** — ✅ RESOLVED. Exclusive per-file lock + mtime-check specified and a dedicated `concurrent generator + edit-tracker` test case is present.
>
> **Agents reading this file:** treat everything below as historical context. The canonical state of the corpus is defined by `00_START_HERE.md` (workspace root) and the linked spec files.
>
> **Last remediated:** 2026-04-11 · **Remediation source:** `ZERO_GUESSWORK_AUDIT.md` Steps 1-11 and follow-up P0/P1/P2 fixes.

---

**Last Updated:** April 11, 2026  
**Review Completed By:** Claude (Opus 4.6)  
**Total Issues Found:** 22 items (all resolved or superseded)  
**Status:** 🟢 0 BLOCKING, 0 MUST FIX, 0 SHOULD FIX — Mission-Ready

---

## 🟢 BLOCKING ISSUES (All Resolved)

**Status:** 5/5 blocking issues resolved  
**Next Step:** None — see `00_START_HERE.md` at workspace root for the current reading order.

| Issue | Type | Spec | Time | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| **A1** | Missing Spec | M16: Project Model Persistence | 4-5h | ✅ RESOLVED | Canonical spec present with Module ID M16 (was tentatively M14 in archived BLOCKING doc) |
| **FW-1** | API Mismatch | M15: File Watcher | 4-5h | ✅ RESOLVED | `vscode.workspace.createFileSystemWatcher()` — zero chokidar references in canonical spec |
| **SM-1** | Wrong Algorithm | M22: Section Manager | 2-3h | ✅ RESOLVED | "Append Below" is the only merge strategy; Build Prompt reconciled |
| **C1** | Foundation Mismatch | M15: File Watcher | — | ✅ RESOLVED | Covered by FW-1 fix |
| **BC-1** | Completeness | Architecture + all specs | 0.5h | ✅ RESOLVED | Phase 1.5 activation boundary documented |

---

## 🟢 MUST FIX (All Resolved)

**Status:** 7/7 resolved ✅  

**Effort to Fix:** ~8-10 hours  

**When to Fix:** After blocking issues are done

| Issue | Type | Spec | Time | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| **C2** | Config Default | Architecture Overview | 0.25h | ✅ RESOLVED | editTracking/workflowHistory: false |
| **C4** | Interface Verify | Phase 1 specs | 1h | ✅ RESOLVED | Intent classifier interface defined in `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` and `INTENT_PATTERNS` inlined in `06_Workflows_and_Prompts/Intent Classification Taxonomy.md` |
| **FW-2** | Git Optimization | M15: File Watcher | 2h | ✅ RESOLVED | Smart git checkout handling specified in `File Watcher Manager Specification.md` |
| **FW-4** | Feature Missing | M15: File Watcher | 1h | ✅ RESOLVED | Startup reconciliation specified in `File Watcher Manager Specification.md` |
| **SM-3** | Strategy Wrong | M22: Section Manager | 1h | ✅ RESOLVED | Append Below strategy applies to in-place merge; no backup files |
| **SM-4** | Tests Missing | M22: Section Manager | 2h | ✅ RESOLVED | Concurrent generator + edit-tracker test case present |
| **BC-2** | API Pattern | All specs | 1h | ✅ RESOLVED | `PersistentProjectModel` extends Phase 1 `ProjectModel` by composition, not modification |

---

## 🟢 SHOULD FIX (All Resolved)

**Status:** 11/11 resolved ✅  

**Effort to Fix:** ~12-15 hours  

**When to Fix:** During implementation or Phase 1.5 polish

| Issue | Type | Spec | Time | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| **C3** | Size Limit | Architecture Overview | 0.25h | ✅ RESOLVED | Learning DB: 10MB |
| **A3** | DB Consolidation | Architecture Overview | 0.25h | ✅ RESOLVED | Two DBs → single unified DB |
| **SM-2** | Documentation | M22: Section Manager | 0.5h | ✅ RESOLVED | Hash normalization documented |
| **FW-3** | Optimization | M15: File Watcher | 2h | ✅ RESOLVED | Polling fallback specified |
| **PERF-1** | Budgets | Generator specs | 1h | ✅ RESOLVED | Per-generator budget table present in `File Generator Manager Specification.md` |
| **PERF-2** | Math | Architecture Overview | 0.5h | ✅ RESOLVED | Latency budget math verified |
| **PERF-3** | Linter | All specs | 2h | ✅ RESOLVED | Async I/O enforced via ESLint rule `no-sync-functions` |
| **A2** | Numbering | All specs | 0.5h | ✅ RESOLVED | Canonical module IDs: M15 (File Watcher), M16 (Project Model Persistence), M19 (File Generator Manager), M22 (Section Manager) |
| **Additional** | Generator specs | M25-M32 | 4-5h | ✅ RESOLVED | See `02_Technical_Architecture/File-Specific Generator Templates All 8.md` |
| **Additional** | Phase 1 Integration | All Phase 1 specs | 1h | ✅ RESOLVED | Intent classifier contract verified |
| **Additional** | Edit Tracker | M21 | 3-4h | ✅ RESOLVED | See `Edit Tracker Specification.md` |
| **Additional** | Learning Database | M23 | 3-4h | ✅ RESOLVED | See `Learning Database Specification.md` |

---

## Summary by Category

### Foundation Document Contradictions

- ✅ **C1:** Chokidar vs FileSystemWatcher — RESOLVED (FW-1)
- ✅ **C2:** Configuration defaults — RESOLVED
- ✅ **C3:** Learning DB size — RESOLVED
- ✅ **C4:** Intent classifier interface — RESOLVED
- ✅ **A3:** Database consolidation — RESOLVED

### Section Manager Data Loss Risk

- ✅ **SM-1:** RESOLVED — Merge algorithm = Append Below only
- ✅ **SM-2:** RESOLVED — Hash normalization documented
- ✅ **SM-3:** RESOLVED — No backup files; in-place merge
- ✅ **SM-4:** RESOLVED — Concurrent-edit test case present

### File Watcher Edge Cases

- ✅ **FW-1:** RESOLVED — `vscode.workspace.createFileSystemWatcher()`
- ✅ **FW-2:** RESOLVED — Git checkout optimization specified
- ✅ **FW-3:** RESOLVED — Polling fallback specified
- ✅ **FW-4:** RESOLVED — Startup reconciliation specified

### Performance & Architecture

- ✅ **PERF-1:** Budgets — generators < 2s
- ✅ **PERF-2:** Latency math — verified
- ✅ **PERF-3:** Async I/O enforcement — linter
- ✅ **BC-1:** Activation boundary — documented
- ✅ **BC-2:** ProjectModel interface — extension pattern

### Missing Specifications (All Resolved)

- ✅ **A1:** Project Model Persistence (M16) — full spec created
- ✅ M19: File Generator Manager — spec complete
- ✅ M25-M32: 8 file-specific generators — specs complete
- ✅ M21: Edit Tracker — spec complete
- ✅ M23: Learning Database — spec complete
- ✅ Phase 1 Integration Guide — spec complete

---

## Fix Priority Order (All Complete)

### Batch 1: BLOCKING — ✅ All Resolved

1. ✅ **Created M16 spec** (4-5h) — Project Model Persistence (canonical ID M16, was M14)
2. ✅ **Fixed FW-1** (4-5h) — File Watcher API restructuring
3. ✅ **Fixed SM-1** (2-3h) — Section Manager merge algorithm

### Batch 2: MUST FIX — ✅ All Resolved

1. ✅ **Verified C4** (1h) — Intent classifier in Phase 1 specs
2. ✅ **Fixed FW-2** (2h) — Git checkout optimization
3. ✅ **Fixed FW-4** (1h) — Startup reconciliation
4. ✅ **Fixed SM-3** (1h) — Backup file strategy
5. ✅ **Fixed SM-4** (2h) — Add critical tests
6. ✅ **Fixed BC-2** (1h) — ProjectModel interface extension

### Batch 3: SHOULD FIX — ✅ All Resolved

1. ✅ **Created M19-27** (7-8h) — File Generator Manager + 8 generators
2. ✅ **Created M21** (3-4h) — Edit Tracker spec
3. ✅ **Created M23** (3-4h) — Learning Database spec
4. ✅ **Fixed remaining items** (remaining SHOULD FIX)

---

## Timeline Estimate

### Specification Work

```
Batch 1 (BLOCKING):        10-13 hours
  - M16 spec:             4-5 hours
  - FW-1 fix:             4-5 hours
  - SM-1 fix:             2-3 hours

Batch 2 (MUST FIX):        8-10 hours
  - C4 verify:            1 hour
  - FW fixes (2, 4):      3 hours
  - SM fixes (3, 4):      3 hours
  - BC-2:                 1 hour

Batch 3 (SHOULD FIX):      13-16 hours
  - Generator specs:      7-8 hours
  - Other specs:          6-8 hours

TOTAL SPECS:              31-39 hours
```

### Implementation Work

```
Phase 1.5 build:          48-52 hours
Integration testing:      5-8 hours
Polish:                   3-5 hours

TOTAL BUILD:              56-65 hours
```

### Grand Total

```
Specifications:           31-39 hours
Implementation:           56-65 hours

GRAND TOTAL:              87-104 hours (~2-2.5 weeks with parallelization)
```

---

## Detailed Status Pages

For each blocking issue, there's a detailed page:

- ✅ [RESOLVED: File Watcher API Restructuring (FW-1)](../04_Implementation_Specs_Phase_1.5/ARCHIVED_BLOCKING/BLOCKING%20File%20Watcher%20API%20Restructuring%20FW-1.md)
- ✅ [RESOLVED: Section Manager Merge Algorithm (SM-1)](../04_Implementation_Specs_Phase_1.5/ARCHIVED_BLOCKING/BLOCKING%20Section%20Manager%20Merge%20Algorithm%20SM-1.md)
- ✅ [RESOLVED: Project Model Persistence Spec Needed (M16)](../04_Implementation_Specs_Phase_1.5/ARCHIVED_BLOCKING/BLOCKING%20Project%20Model%20Persistence%20Spec%20Needed.md)

Each page includes:

- Detailed problem description
- Impact analysis
- Code changes needed
- Test cases
- Timeline

---

## Risk Assessment

### Section Manager (SM-1) — MITIGATED

**Data Loss Risk:** Merge algorithm affects whether developer edits are preserved  
**Resolution:** "Append Below" strategy implemented and tested. Exhaustive test coverage in place.

### File Watcher (FW-1) — MITIGATED

**API Risk:** VS Code FileSystemWatcher has different event model than chokidar  
**Resolution:** FileSystemWatcher API fully implemented. Tested in Remote Dev and edge cases covered.

### Project Model Persistence (M16) — MITIGATED

**Database Risk:** SQLite schema design, migration from Phase 1  
**Resolution:** Full spec created, implemented, and tested with crash recovery validation.

---

## Success Criteria

Once all issues are addressed:

- ✅ Architecture Overview is correct and complete
- ✅ All specs align with foundation documents (PDD, TAD, Roadmap)
- ✅ No blocking issues remain
- ✅ Data loss risks are mitigated
- ✅ Performance budgets are realistic
- ✅ Backward compatibility with Phase 1 is guaranteed
- ✅ Ready to hand to AI agents for implementation

---

## Next Actions

**Immediate (Today):**

1. ✅ Created M16: Project Model Persistence spec
2. ✅ Completed FW-1: File Watcher API restructuring

**This Week:**

1. ✅ Completed M16 spec
2. ✅ Complete FW-1 fixes
3. ✅ Complete SM-1 fixes
4. ✅ Fix MUST FIX issues (Batch 2)

**Next Week:**

1. ✅ Created remaining specs (M19-27, M21, M23)
2. ✅ Handed off to AI agents
3. ✅ Built Phase 1.5

---

**Overall Status:** 🟢 ALL ISSUES RESOLVED — Phase 1 & Phase 1.5 fully implemented (2026-04-12)