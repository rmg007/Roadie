# 🟢 Additional Issues: SHOULD FIX & Nice-to-Haves

> ## ⚠️ SUPERSEDED — HISTORICAL REFERENCE
>
> This file was a pre-remediation list of nice-to-have polish items. **Every item that was a spec gap is now resolved** (see `Issues & Fixes Status Tracking Dashboard.md` — all rows show ✅ RESOLVED).
>
> Items related to the "M14 vs M16" module numbering dispute are no longer relevant — the canonical module ID is **M16** (see `00_START_HERE.md` at workspace root and `Project Model Persistence Specification.md:9`).
>
> **Agents:** do NOT treat this file as a current task list. Use `00_START_HERE.md` as the entry point.
>
> **Superseded on:** 2026-04-11

**Status:** ✅ ALL ITEMS RESOLVED or DEFERRED (as of 2026-04-11)  
**When to Fix:** N/A — superseded document.

---

## Documentation & Design Issues

### SM-2: Hash Normalization Behavior 🟢 SHOULD FIX

**Issue:** Section Manager spec uses normalized hashing (whitespace trim + empty line filter). Developers may not expect this behavior.

**Current Spec:**

```tsx
const normalized = content
  .split('\n')
  .map(line => line.trim())
  .filter(line => line.length > 0)
  .join('\n');
const hash = sha256(normalized).substring(0, 8);
```

**Problem:** Developer adds only blank lines or changes indentation → hash doesn't change → Roadie overwrites formatting.

**Fix:** Document this behavior clearly in spec. Consider optional `roadie.strictEditDetection` config for developers who want formatting changes preserved.

**Time:** 0.5 hours (documentation)

---

### A2: Module Numbering Inconsistency 🟢 SHOULD FIX

**Issue:** Roadmap uses M0-M23, Phase 1.5 specs use different numbers (M14, M15, M22).

**Example:**

- Roadmap: M16 = "Automatic File Regeneration" (should be Section Manager)
- Phase 1.5 spec: M22 = Section Manager
- These don't align

**Fix:** Add mapping table to Phase 1.5 Master Index:

| Roadmap | Phase 1.5 Spec | Module |
| --- | --- | --- |
| M14 | M14 | Project Model Persistence |
| M15 | M15 | File Watcher Manager |
| M16 | M22 | Section Manager (Automatic Generation) |
| ... | ... | ... |

**Time:** 0.5 hours (add mapping table)

---

### PERF-3: Async I/O Enforcement 🟢 SHOULD FIX

**Issue:** Section Manager pseudocode uses `fs.readFileSync()`. Synchronous file I/O blocks VS Code extension host and freezes editor.

**Current Risk:**

```tsx
// DON'T DO THIS - blocks extension host
const content = fs.readFileSync('.github/file.md', 'utf-8');
```

**Fix:** Add to Implementation Patterns page:

```
## Critical Rule: All Phase 1.5 I/O Must Be Async

All file I/O operations in Phase 1.5 (watcher/, generators/, persistence/) 
MUST use async APIs: fs.promises.readFile, fs.promises.writeFile.

Synchronous operations (fs.readFileSync, fs.writeFileSync, fs.statSync) are 
FORBIDDEN in these directories. Violations block extension host and freeze VS Code.

Linter check: Reject synchronous fs calls in phase-1.5 directories.
```

**Time:** 2 hours (linter setup + enforcement)

---

## Performance & Optimization Issues

### FW-3: Polling Fallback Optimization 🟢 SHOULD FIX

**Current:** Polls entire workspace every 5 seconds when FileSystemWatcher fails.

**Problem:** Large projects (10,000+ files) means expensive I/O every 5 seconds.

**Fix:** Smart polling:

- Only poll watched directories (dependencies, config files)
- Source code: 30s polling interval (structure changes rare)
- Config files: 5s interval (important for generators)
- Skip node_modules, .git, build output entirely

**Expected Impact:**

- Reduces I/O by 80-90% for large projects
- No behavioral change

**Time:** 2 hours (File Watcher spec update + testing)

---

### PERF-2: Latency Budget Math Verification 🟢 SHOULD FIX

**Current Budgets:**

```
Total event latency: < 1s
  - Watcher debounce: 500ms
  - Classification: 5ms
  - Project model update: 50ms
  - Generator trigger: 50ms
  ___________________
  Total: 605ms (OK)
```

**But Under Load:** Generator dispatch might take 200-500ms if multiple generators triggered. Math might exceed budget.

**Fix:** Adjust budgets:

- Reduce debounce to 300ms OR
- Increase total budget to 1.5s OR
- Parallelize generator execution

**Recommendation:** Test under realistic load and adjust based on measurements.

**Time:** 1 hour (measurement + adjustment)

---

## Testing & Verification

### SM-4: Missing Critical Test Cases 🟠 MUST FIX (moved to BLOCKING)

Actually should be MUST FIX, not SHOULD FIX. Already covered in BLOCKING section.

---

### PERF-1: Generator Timing Assertions 🟢 SHOULD FIX

**Current:** Architecture says "< 2s all generators combined" but tests don't enforce this.

**Fix:** Add to every generator integration test:

```tsx
it('runs in under 2 seconds', async () => {
  const startTime = performance.now();
  await runAllGenerators(projectModel);
  const elapsed = performance.now() - startTime;
  expect(elapsed).toBeLessThan(2000);
});
```

**Time:** 1 hour (add to all generator tests)

---

## API & Architecture Issues

### BC-2: ProjectModel Interface Extension Pattern 🟡 MUST FIX

Already covered in MUST FIX section.

---

### C4: Intent Classifier Interface Verification 🟠 MUST FIX (moved to BLOCKING)

Actually higher priority than SHOULD FIX. Need to verify Phase 1 specs.

---

## Missing Specifications (Large Effort)

### Missing Generator Specs (M20-27) 🟢 SHOULD FIX

**What:** 8 file-specific generator specifications not created yet

**Includes:**

- M20: Copilot Instructions Generator
- M21: Path Instructions Generators (per-directory)
- M22: Agent Definition Generator
- M23: Skill Generator
- M24: Hooks Generator
- M25: Workflows Generator
- M26: Templates Generator
- M27: [AGENTS.md](http://AGENTS.md) Generator

**Pattern:** Each follows same template:

1. Input (project model queries)
2. Template (Markdown/YAML template)
3. Output (file path, format)
4. Triggers (what changes trigger regeneration)
5. Tests (unit + integration)

**Time:** 7-8 hours total (1 hour per spec)

**Can Parallelize:** Once M19 (File Generator Manager) spec is done, these can be written in parallel.

---

### Missing M21: Edit Tracker Spec 🟢 SHOULD FIX

**What:** How to detect, diff, and track human edits to generated files.

**Includes:**

- Diff computation (before/after snapshots)
- Change detection (what changed)
- Snapshot storage (SQLite)
- Integration with File Watcher

**Time:** 3-4 hours

---

### Missing M23: Learning Database Spec 🟢 SHOULD FIX

**What:** SQLite schema and queries for learning system.

**Tables:**

- edit_history (developer edits over time)
- workflow_outcomes (successes/failures)
- discovered_patterns (conventions learned)

**Time:** 3-4 hours

---

### Missing Phase 1 Integration Guide 🟢 SHOULD FIX

**What:** How Phase 1.5 extends Phase 1 without breaking it.

**Includes:**

- Workflow Engine changes (outcome logging)
- Agent Spawner changes (richer context)
- ProjectModel interface extension
- Migration guide (Phase 1 → Phase 1.5)

**Time:** 2-3 hours

---

## Effort Summary

| Issue | Time | Priority |
| --- | --- | --- |
| SM-2 (hash normalization docs) | 0.5h | SHOULD FIX |
| A2 (module numbering) | 0.5h | SHOULD FIX |
| PERF-3 (async I/O linter) | 2h | SHOULD FIX |
| FW-3 (polling optimization) | 2h | SHOULD FIX |
| PERF-2 (latency math) | 1h | SHOULD FIX |
| PERF-1 (timing assertions) | 1h | SHOULD FIX |
| Generator specs (M20-27) | 7-8h | SHOULD FIX |
| Edit Tracker spec (M21) | 3-4h | SHOULD FIX |
| Learning Database spec (M23) | 3-4h | SHOULD FIX |
| Phase 1 Integration guide | 2-3h | SHOULD FIX |
| **TOTAL** | **22-25h** |  |

---

## When to Schedule

**Batch 1 (BLOCKING):** This week — 10-13 hours

**Batch 2 (MUST FIX):** Next few days — 8-10 hours

**Batch 3 (SHOULD FIX):** Next week or during build — 22-25 hours

**Total Timeline:** ~40-48 hours to completion

---

## Quick Wins (Easy Fixes)

These can be done in parallel or as breaks between larger work:

1. **SM-2:** Documentation (0.5h)
2. **A2:** Mapping table (0.5h)
3. **PERF-1:** Test assertions (1h)

These three can be done in 2 hours and provide good value.

---

## Deferred to Phase 1.5 Polish

Can be deferred if needed:

- Generator specs (can use template-based AI generation)
- Edit Tracker spec (lower risk module)
- Learning Database spec (lower risk module)
- Phase 1 Integration guide (can write during implementation)
- Polling optimization (nice-to-have, not critical)
- Async I/O linter (can add after basic build works)

**Minimum to Build:** Just the BLOCKING + MUST FIX issues (~18-23 hours)

---

## Status

✅ Issues identified  

✅ Effort estimated  

⏳ Scheduling pending  

🔴 All items deferred until after BLOCKING + MUST FIX batches