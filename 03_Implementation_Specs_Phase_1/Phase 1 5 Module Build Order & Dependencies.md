# 📊 Phase 1.5 Module Build Order & Dependencies

## Phase 1.5 Build Sequence, Dependencies, Verification Criteria

---

## Build Principles

1. **Phase 1 must pass all tests at every step** — backward compatibility is non-negotiable
2. **Build the foundation first** — Project Model Persistence is the blocker for everything
3. **Section Manager gets extra time** — it's the highest-risk module (data loss potential)
4. **Generators follow a template** — once the first generator works, the rest are mechanical
5. **Integration test after each milestone** — verify the full pipeline works

---

## Build Order

### Step 1: Project Model Persistence (M16) — BLOCKER

**Estimated time:** 5-6 hours

**Depends on:** Phase 1 complete

**Creates:** `src/model/project-model-persistence.ts`

**Verification:**

- Model loads from SQLite in <500ms
- Model reconciles with file system on activation
- Incremental updates work (applyFileChange)
- Debounced writes flush every 5s
- Phase 1 tests still pass
- `isPhase15Active()` returns true when DB exists and model is populated

### Step 2: Learning Database (M20)

**Estimated time:** 3-4 hours

**Depends on:** M16 (shared SQLite connection)

**Creates:** `src/learning/learning-database.ts`

**Verification:**

- Schema migration adds tables to existing DB
- Snapshot CRUD works
- Workflow history CRUD works (when enabled)
- Section hash CRUD works
- Pruning removes old entries correctly
- DB stays under 10 MB after 200 workflow runs

### Step 3: File Watcher Manager (M15)

**Estimated time:** 4-5 hours

**Depends on:** M16 (dispatches to model updater)

**Creates:** `src/watcher/file-watcher-manager.ts`, `src/watcher/change-classifier.ts`

**Verification:**

- VS Code FileSystemWatcher detects package.json changes
- Debounce batches rapid changes
- Classification routes to correct handler
- Large batch (>1000 events) triggers full rescan
- Startup reconciliation catches changes made while VS Code was closed
- Watcher error doesn't crash extension

### Step 4: Section Manager (M22) — CRITICAL RISK

**Estimated time:** 6-8 hours

**Depends on:** M20 (stores section hashes)

**Creates:** `src/generator/section-manager.ts`, `src/generator/section-parser.ts`, `src/generator/merge-algorithm.ts`

**Verification:**

- Parses Markdown section markers correctly
- Hash comparison detects human edits
- **Append-below merge:** human-edited section gets new content appended below with `<!-- roadie:merged:timestamp -->`
- Unmodified sections are replaced entirely
- Deleted markers → file is human-owned, skip with warning
- First-run (no markers in existing file) → new sections appended at bottom
- Moved sections found by ID, not position
- **40+ unit tests pass**
- **5+ integration tests with real file system pass**
- **Zero data loss in any scenario** (human edits never discarded)

### Step 5: File Generator Manager (M19)

**Estimated time:** 4-5 hours

**Depends on:** M16, M20, M22

**Creates:** `src/generator/file-generator-manager.ts`, `src/generator/deferred-write-queue.ts`

**Verification:**

- Subscribes to model change events
- Maps change types to correct generators
- Runs generators in parallel (<2s total)
- Defers writes when files are open in editor
- Logs generation events to Learning Database
- `.github/.roadie/.gitignore` created

### Step 6: Edit Tracker (M21)

**Estimated time:** 3-4 hours

**Depends on:** M15 (receives USER_EDIT events), M20, M22

**Creates:** `src/tracking/edit-tracker.ts`

**Verification:**

- Only active when `roadie.editTracking` is true
- Detects edits inside Roadie sections
- Detects content added outside markers
- Stores snapshots with source 'human'
- Returns null when tracking is disabled

### Step 7: Copilot Instructions Generator (first generator)

**Estimated time:** 2-3 hours

**Depends on:** M19 (registered as generator sub-module), M22

**Creates:** `src/generator/templates/copilot-instructions.ts`

**Verification:**

- Generates `.github/copilot-instructions.md` with tech stack, conventions, commands
- Uses Roadie section markers
- Content is accurate for test fixture projects
- Hash comparison prevents unnecessary writes

### Step 8: [AGENTS.md](http://AGENTS.md) Generator

**Estimated time:** 1-2 hours

**Depends on:** M19

**Creates:** `src/generator/templates/agents-md.ts`

**Verification:**

- Generates `AGENTS.md` at project root
- Contains project overview and cross-tool instructions

### Step 9: Remaining 6 Generators (parallel work)

**Estimated time:** 6-8 hours total (1-2 hours each)

**Depends on:** M19

**Creates:** Path Instructions, Agent Definitions, Skills, Hooks, Workflows, Templates generators

**Verification:**

- Each generator produces correct file content
- All use section markers
- All <250ms per generation

### Step 10: Phase 1 Integration

**Estimated time:** 3-4 hours

**Depends on:** All above

**Modifies:** `extension.ts`, `types.ts`, `database.ts`

**Verification:**

- Extension activates with Phase 1.5 when DB exists
- Extension falls back to Phase 1 when DB doesn't exist
- Workflow outcomes logged to learning DB
- All Phase 1 tests pass
- All Phase 1.5 tests pass
- Full end-to-end: edit package.json → model updates → files regenerate → sections preserved

---

## Dependency Graph

```
Phase 1 (complete)
    │
    ▼
M16: Project Model Persistence  ────────────────────────┐
    │                                                   │
    ├───────────────────────┐                        │
    ▼                        ▼                        ▼
M20: Learning DB          M15: File Watcher    M19: Generator Mgr
    │                        │                        │
    ▼                        │                        │
M22: Section Manager  ─────┴──────────────────────┘
    │
    ├───────────────────────┐
    ▼                        ▼
M21: Edit Tracker      Generators (8 modules)
                             │
                             ▼
                    Phase 1 Integration
```

---

## Total Estimated Time

| Step | Module | Hours |
| --- | --- | --- |
| 1 | Project Model Persistence | 5-6 |
| 2 | Learning Database | 3-4 |
| 3 | File Watcher Manager | 4-5 |
| 4 | Section Manager (CRITICAL) | 6-8 |
| 5 | File Generator Manager | 4-5 |
| 6 | Edit Tracker | 3-4 |
| 7-9 | 8 Generator sub-modules | 8-10 |
| 10 | Phase 1 Integration | 3-4 |
| **Total** |  | **36-46 hours** |

---

## Risk Mitigation Checkpoints

**After Step 1:** Can the model persist and load? → If not, everything stops.

**After Step 4:** Does Section Manager preserve human edits in ALL scenarios? → If not, generators will cause data loss.

**After Step 5:** Does the full pipeline work (model change → generators → file write)? → This is the first end-to-end validation.

**After Step 10:** Does Phase 1 still work perfectly without Phase 1.5 features? → Backward compatibility gate.