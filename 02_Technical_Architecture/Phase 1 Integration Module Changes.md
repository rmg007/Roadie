# 🔄 Phase 1 Integration: Module Changes

## Exactly What Changes in Phase 1 Modules for Phase 1.5 — Zero Breaking Changes

---

## Guiding Principle

**Phase 1 code must work perfectly without Phase 1.5.** All Phase 1.5 changes are additive. No existing method signatures change. No existing behavior changes. Phase 1.5 modules extend, wrap, or subscribe — they never modify Phase 1 internals.

---

## Module-by-Module Changes

### 1. extension.ts — Activation Sequence Changes

**Phase 1 activation:**

1. Register Chat Participant
2. Create ProjectModel (empty, in-memory)
3. Register status bar
4. Register commands

**Phase 1.5 additions (after Phase 1 activation completes):**

```tsx
// In activate(), AFTER all Phase 1 initialization:

if (isPhase15Available()) {
  // 5. Upgrade to PersistentProjectModel
  const persistentModel = new PersistentProjectModelImpl(phase1Model);
  await persistentModel.loadFromDb();
  await persistentModel.reconcileWithFileSystem();
  container.replace('projectModel', persistentModel);
  
  // 6. Start File Watcher
  const watcher = new FileWatcherManager();
  await watcher.start({ workspace: workspaceRoot });
  watcher.on('file-change', (change) => {
    if (change.classifiedAs === 'USER_EDIT') {
      editTracker.trackEdit(change.filePath);
    } else {
      persistentModel.applyFileChange(change);
    }
  });
  
  // 7. Initialize File Generator Manager
  const generatorManager = new FileGeneratorManager();
  generatorManager.initialize(persistentModel);
  
  // 8. Initialize Edit Tracker (if opt-in)
  const editTracker = new EditTracker();
  editTracker.initialize(config);
  
  // 9. Run initial pruning
  await learningDb.prune();
}
```

**Key:** `isPhase15Available()` checks if the Phase 1.5 modules exist in the bundle. This allows shipping Phase 1 without Phase 1.5 code.

**Deactivation additions:**

```tsx
// In deactivate(), BEFORE Phase 1 cleanup:
if (persistentModel) {
  await persistentModel.saveToDb(); // Flush pending writes
}
if (watcher) {
  await watcher.stop();
}
```

---

### 2. ProjectModel — Interface Extension (NOT Modification)

**Phase 1 interface (UNCHANGED):**

```tsx
interface ProjectModel {
  getTechStack(): TechStackEntry[];
  getDirectoryStructure(): DirectoryNode;
  getPatterns(): DetectedPattern[];
  getPreferences(): DeveloperPreferences;
  getCommands(): ProjectCommand[];
  toContext(options?: ContextOptions): ProjectContext;
  update(delta: ProjectModelDelta): void;
}
```

**Phase 1.5 extension (NEW):**

```tsx
interface PersistentProjectModel extends ProjectModel {
  loadFromDb(): Promise<void>;
  saveToDb(): Promise<void>;
  reconcileWithFileSystem(): Promise<ReconciliationResult>;
  applyFileChange(change: ClassifiedFileChange): Promise<void>;
  isPopulated(): boolean;
  getLastAnalyzedAt(): Date | null;
  onModelChanged(listener: (delta: ProjectModelDelta) => void): Disposable;
}
```

**Implementation note:** `PersistentProjectModelImpl` wraps the Phase 1 `ProjectModelImpl`. It delegates all Phase 1 methods to the wrapped instance. It adds persistence, incremental updates, and change events on top.

```tsx
class PersistentProjectModelImpl implements PersistentProjectModel {
  constructor(private inner: ProjectModel) {}
  
  // Delegate Phase 1 methods
  getTechStack() { return this.inner.getTechStack(); }
  getDirectoryStructure() { return this.inner.getDirectoryStructure(); }
  toContext(options?) { return this.inner.toContext(options); }
  update(delta) {
    this.inner.update(delta);
    this.markDirty();
    this.emitModelChanged(delta);
  }
  
  // Phase 1.5 additions
  async loadFromDb() { /* ... */ }
  async saveToDb() { /* ... */ }
  // ...
}
```

---

### 3. Workflow Engine — Outcome Logging

**What changes:** After a workflow completes, log the outcome to the Learning Database.

**How (non-breaking):** Add an optional `onWorkflowComplete` hook in the container, not in the WorkflowEngine interface.

```tsx
// In extension.ts, Phase 1.5 activation:
workflowEngine.onComplete = async (result: WorkflowResult) => {
  await learningDb.recordWorkflowOutcome({
    workflowType: result.workflowId,
    prompt: result.originalPrompt,
    status: result.state === 'COMPLETED' ? 'completed' : 'failed',
    stepsCompleted: result.stepResults.filter(s => s.status === 'success').length,
    stepsTotal: result.stepResults.length,
    durationMs: result.duration,
    modelTiersUsed: result.modelTiersUsed,
    errorSummary: result.state === 'FAILED' ? result.error : undefined
  });
};
```

**Why this approach:** The WorkflowEngine already has `execute()` which returns a `WorkflowResult`. We hook into the result AFTER execution, not during. No changes to the engine's internal logic.

---

### 4. Agent Spawner — Richer Context Injection

**What changes:** When Phase 1.5 is active, the project model has more context (persistent state, detected patterns, learned preferences). The Agent Spawner doesn't change — it already calls `projectModel.toContext()` and injects the result. The richer context comes automatically from the PersistentProjectModel.

**No code changes needed.** The Agent Spawner is already designed to work with whatever `toContext()` returns.

---

### 5. File Generator (Phase 1) — Subsumed by File Generator Manager

**Phase 1 file generator** (`generator/file-generator.ts`) produces `copilot-instructions.md` and `AGENTS.md`. In Phase 1.5, the **File Generator Manager** takes over orchestration.

**Migration approach:**

- Phase 1's `file-generator.ts` becomes one of the 8 generator sub-modules
- Its `generate()` method is unchanged
- The File Generator Manager calls it instead of the Chat Participant calling it directly
- Phase 1's `section-manager.ts` is extended (not replaced) by the Phase 1.5 Section Manager

```tsx
// Phase 1: Chat Participant calls file generator directly
if (workflowComplete) {
  await fileGenerator.generate('copilot_instructions');
  await fileGenerator.generate('agents_md');
}

// Phase 1.5: File Generator Manager subscribes to model changes
// and calls generators automatically. The Chat Participant no longer
// calls generators directly.
// BUT: if Phase 1.5 is not active, Phase 1 behavior is preserved.
if (isPhase15Active(projectModel)) {
  // Generator Manager handles it via model change events
} else {
  // Phase 1 behavior: manual generation after workflow
  await fileGenerator.generate('copilot_instructions');
  await fileGenerator.generate('agents_md');
}
```

---

### 6. Database (database.ts) — Schema Migration

**What changes:** Phase 1.5 adds new tables to the same database.

**Migration strategy:** On activation, check `schema_version` table. If version < 2, run Phase 1.5 migration.

```tsx
function migrateToPhase15(db: BetterSqlite3.Database): void {
  const version = db.prepare('SELECT MAX(version) as v FROM schema_version').get();
  
  if (version.v < 2) {
    db.transaction(() => {
      // Add Phase 1.5 tables
      db.exec(`
        CREATE TABLE IF NOT EXISTS file_snapshots (...);
        CREATE TABLE IF NOT EXISTS workflow_history (...);
        CREATE TABLE IF NOT EXISTS section_hashes (...);
        INSERT INTO schema_version (version) VALUES (2);
      `);
    })();
  }
}
```

**Key:** Migration is idempotent (`CREATE TABLE IF NOT EXISTS`). Running it multiple times is safe.

---

## Verification Checklist

After Phase 1.5 integration, verify:

- [ ]  All Phase 1 tests still pass (`npm run test`)
- [ ]  Extension activates without Phase 1.5 features (delete `.github/.roadie/project-model.db`)
- [ ]  `@roadie` Chat Participant works identically to Phase 1
- [ ]  All 7 workflows execute correctly
- [ ]  Intent classification is unchanged
- [ ]  Model escalation works as before
- [ ]  No new settings are enabled by default
- [ ]  Extension deactivates cleanly in both Phase 1 and Phase 1.5 modes

---

## Summary: What's Changed vs What's New

| Module | Changed? | How |
| --- | --- | --- |
| extension.ts | **Modified** | Additional activation/deactivation steps (guarded by isPhase15Active) |
| types.ts | **Extended** | New PersistentProjectModel interface added (Phase 1 interfaces untouched) |
| project-model.ts | **Wrapped** | PersistentProjectModelImpl wraps Phase 1 implementation |
| workflow-engine.ts | **Hooked** | onComplete callback added externally (no internal changes) |
| agent-spawner.ts | **Unchanged** | Automatically benefits from richer toContext() |
| file-generator.ts | **Subsumed** | Becomes a sub-module of File Generator Manager |
| database.ts | **Migrated** | New tables added via schema migration |
| intent-classifier.ts | **Unchanged** | No Phase 1.5 dependencies |
| step-executor.ts | **Unchanged** | No Phase 1.5 dependencies |
| model-resolver.ts | **Unchanged** | No Phase 1.5 dependencies |