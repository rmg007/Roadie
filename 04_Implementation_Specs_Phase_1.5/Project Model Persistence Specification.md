# 💾 Project Model Persistence Specification

## Extends Phase 1 In-Memory Model with SQLite Persistence, Incremental Updates, Startup Reconciliation

---

## Module Identity

**Module ID:** M16

> **CANONICAL MODULE ID:** M16. This is the authoritative number for the Project Model Persistence module across the entire corpus. Any historical references to M14 (in archived BLOCKING files) are obsolete.

**File Location:** `src/model/project-model-persistence.ts`

**Depends On:** Phase 1 Project Model (`src/model/project-model.ts`), Database (`src/model/database.ts`)

**Used By:** File Watcher Manager, All File Generators, Workflow Engine (enhanced context)

**Complexity:** Medium (startup reconciliation logic requires careful design)

**Estimated Build Time:** 5-6 hours

**Implementation Status:** ✅ COMPLETE — Implemented as of 2026-04-12

---

## The Problem

Phase 1's project model is rebuilt from scratch on every VS Code session. This works but is slow (full scan on every activation) and loses learned context. Phase 1.5 needs:

1. **Persist** the model to SQLite so it survives across sessions
2. **Load** from SQLite at activation (fast: <500ms)
3. **Validate** against the current file system (detect out-of-sync state)
4. **Update incrementally** when the file watcher reports changes (not full rescan)
5. **Maintain backward compatibility** — Phase 1 code that uses `ProjectModel` must work unchanged

---

## Interface Extension Pattern

> **Critical:** Phase 1's `ProjectModel` interface is NOT modified. Phase 1.5 EXTENDS it.
> 

```tsx
// Phase 1 interface (UNCHANGED — lives in types.ts)
interface ProjectModel {
  getTechStack(): TechStackEntry[];
  getDirectoryStructure(): DirectoryNode;
  getPatterns(): DetectedPattern[];
  getPreferences(): DeveloperPreferences;
  getCommands(): ProjectCommand[];
  toContext(options?: ContextOptions): ProjectContext;
  update(delta: ProjectModelDelta): void;
}

// Phase 1.5 extension (NEW — lives in types.ts)
interface PersistentProjectModel extends ProjectModel {
  /** Load model state from SQLite. Called at activation. */
  loadFromDb(): Promise<void>;
  
  /** Flush pending changes to SQLite. Called at deactivation and periodically. */
  saveToDb(): Promise<void>;
  
  /** Compare model state against current file system. Fix discrepancies. */
  reconcileWithFileSystem(): Promise<ReconciliationResult>;
  
  /** Apply incremental update from file watcher event. */
  applyFileChange(change: ClassifiedFileChange): Promise<void>;
  
  /** Check if the model has been populated (vs empty/first-run). */
  isPopulated(): boolean;
  
  /** Get timestamp of last successful analysis. */
  getLastAnalyzedAt(): Date | null;
  
  /** Subscribe to model change events (used by generators). */
  onModelChanged(listener: (delta: ProjectModelDelta) => void): Disposable;
}

interface ReconciliationResult {
  status: 'in-sync' | 'reconciled' | 'rebuilt';
  changesDetected: number;
  categoriesUpdated: string[];
  durationMs: number;
}

interface ClassifiedFileChange {
  filePath: string;
  eventType: 'create' | 'change' | 'delete';
  classifiedAs: 'DEPENDENCY_CHANGE' | 'CONFIG_CHANGE' | 'STRUCTURE_CHANGE' | 'SOURCE_ADDITION' | 'USER_EDIT' | 'OTHER';
  timestamp: Date;
}
```

---

## Startup Sequence

```tsx
async function initializePersistentModel(): Promise<PersistentProjectModel> {
  const model = new PersistentProjectModelImpl();
  
  // Step 1: Check if database exists
  const dbExists = await fileExists('.github/.roadie/project-model.db');
  
  if (!dbExists) {
    // First run or database deleted — fall back to Phase 1 behavior
    // Model starts empty, will be populated by first workflow
    logger.info('No existing model database. Starting fresh (Phase 1 mode).');
    return model;
  }
  
  // Step 2: Load from SQLite
  try {
    await model.loadFromDb();
    logger.info(`Model loaded from SQLite: ${model.getTechStack().length} stack entries`);
  } catch (error) {
    // Database corrupted — rebuild
    logger.warn(`Database corrupted: ${error.message}. Rebuilding.`);
    await deleteFile('.github/.roadie/project-model.db');
    return model; // Empty model, Phase 1 behavior
  }
  
  // Step 3: Reconcile with file system
  const result = await model.reconcileWithFileSystem();
  
  if (result.status === 'rebuilt') {
    logger.info('Model rebuilt from scratch (major file system changes).');
  } else if (result.status === 'reconciled') {
    logger.info(`Model reconciled: ${result.changesDetected} changes in ${result.durationMs}ms`);
  } else {
    logger.info(`Model in sync (validated in ${result.durationMs}ms)`);
  }
  
  return model;
}
```

---

## Reconciliation Algorithm

```tsx
async function reconcileWithFileSystem(): Promise<ReconciliationResult> {
  const startTime = performance.now();
  const changes: string[] = [];
  
  // 1. Check dependency files (package.json, etc.)
  const currentDeps = await scanDependencyFiles();
  const storedDeps = this.getDependencies();
  if (hashChanged(currentDeps, storedDeps)) {
    await this.updateDependencies(currentDeps);
    changes.push('dependencies');
  }
  
  // 2. Check config files (tsconfig.json, eslint, etc.)
  const currentConfig = await scanConfigFiles();
  const storedConfig = this.getConfigState();
  if (hashChanged(currentConfig, storedConfig)) {
    await this.updateConfig(currentConfig);
    changes.push('config');
  }
  
  // 3. Check directory structure (new/removed directories)
  const currentStructure = await scanDirectoryStructure();
  const storedStructure = this.getDirectoryStructure();
  if (structureChanged(currentStructure, storedStructure)) {
    await this.updateStructure(currentStructure);
    changes.push('structure');
  }
  
  // 4. If >50% of checks changed, do a full rebuild instead
  if (changes.length > 2) {
    await this.fullRebuild();
    return {
      status: 'rebuilt',
      changesDetected: changes.length,
      categoriesUpdated: changes,
      durationMs: performance.now() - startTime
    };
  }
  
  return {
    status: changes.length > 0 ? 'reconciled' : 'in-sync',
    changesDetected: changes.length,
    categoriesUpdated: changes,
    durationMs: performance.now() - startTime
  };
}
```

---

## Incremental Updates (from File Watcher)

```tsx
async function applyFileChange(change: ClassifiedFileChange): Promise<void> {
  switch (change.classifiedAs) {
    case 'DEPENDENCY_CHANGE':
      // Re-read the specific dependency file
      const deps = await parseDependencyFile(change.filePath);
      this.update({ techStack: deps.techStack, commands: deps.commands });
      this.emitModelChanged({ techStack: deps.techStack });
      break;
      
    case 'CONFIG_CHANGE':
      // Re-read the specific config file
      const config = await parseConfigFile(change.filePath);
      this.update({ techStack: config.additions });
      this.emitModelChanged({ techStack: config.additions });
      break;
      
    case 'STRUCTURE_CHANGE':
      // Re-scan directory structure
      const structure = await scanDirectoryStructure();
      this.update({ directories: [structure] });
      this.emitModelChanged({ directories: [structure] });
      break;
      
    default:
      // No model update needed
      break;
  }
  
  // Mark model as dirty (pending SQLite flush)
  this.markDirty();
}
```

---

## Debounced SQLite Writes

```tsx
// Writes are batched every 5 seconds maximum
private flushTimer: NodeJS.Timeout | null = null;
private isDirty = false;

private markDirty(): void {
  this.isDirty = true;
  
  if (!this.flushTimer) {
    this.flushTimer = setTimeout(async () => {
      await this.saveToDb();
      this.isDirty = false;
      this.flushTimer = null;
    }, 5000); // 5 second debounce
  }
}

async deactivate(): Promise<void> {
  // Flush on shutdown
  if (this.isDirty) {
    if (this.flushTimer) clearTimeout(this.flushTimer);
    await this.saveToDb();
  }
}
```

---

## SQLite Operations

Uses the existing `database.ts` module from Phase 1. Key operations:

```tsx
async loadFromDb(): Promise<void> {
  const db = this.database;
  
  // Load tech stack
  this.techStack = db.prepare(
    'SELECT category, name, version, source_file FROM tech_stack'
  ).all() as TechStackEntry[];
  
  // Load directory structure (flatten then rebuild tree)
  const rows = db.prepare(
    'SELECT path, type, language, last_scanned FROM directory_structure'
  ).all();
  this.directoryTree = buildDirectoryTree(rows);
  
  // Load patterns
  this.patterns = db.prepare(
    'SELECT category, description, evidence, confidence FROM detected_patterns'
  ).all().map(row => ({
    ...row,
    evidence: JSON.parse(row.evidence as string)
  })) as DetectedPattern[];
  
  // Load preferences
  const prefs = db.prepare('SELECT key, value FROM developer_preferences').all();
  this.preferences = new Map(prefs.map(p => [p.key, JSON.parse(p.value as string)]));
}

async saveToDb(): Promise<void> {
  const db = this.database;
  
  db.transaction(() => {
    // Upsert tech stack
    const upsertStack = db.prepare(
      'INSERT OR REPLACE INTO tech_stack (category, name, version, source_file, detected_at) VALUES (?, ?, ?, ?, datetime("now"))'
    );
    for (const entry of this.techStack) {
      upsertStack.run(entry.category, entry.name, entry.version, entry.sourceFile);
    }
    
    // Upsert directory structure (flatten tree)
    const upsertDir = db.prepare(
      'INSERT OR REPLACE INTO directory_structure (path, type, language, last_scanned) VALUES (?, ?, ?, datetime("now"))'
    );
    for (const node of flattenTree(this.directoryTree)) {
      upsertDir.run(node.path, node.type, node.language);
    }
    
    // Upsert patterns
    const upsertPattern = db.prepare(
      'INSERT OR REPLACE INTO detected_patterns (category, description, evidence, confidence, detected_at) VALUES (?, ?, ?, ?, datetime("now"))'
    );
    for (const pattern of this.patterns) {
      upsertPattern.run(pattern.category, pattern.description, JSON.stringify(pattern.evidence), pattern.confidence);
    }
  })();
}
```

---

## isPhase15Active() Check

Other Phase 1.5 modules use this to determine whether to activate:

```tsx
function isPhase15Active(model: ProjectModel): model is PersistentProjectModel {
  return 'loadFromDb' in model && (model as PersistentProjectModel).isPopulated();
}
```

This returns `true` only when:

1. The model is a `PersistentProjectModel` instance (not a plain Phase 1 `ProjectModel`)
2. The model has been populated (database exists and was loaded)

If `false`, Phase 1.5 features (file watcher, generators, edit tracker) do NOT initialize.

---

## Testing Strategy

### Unit Tests

```tsx
describe('PersistentProjectModel', () => {
  describe('loadFromDb', () => {
    it('loads tech stack from SQLite', async () => {
      // Pre-populate test database
      const db = createTestDb();
      db.exec("INSERT INTO tech_stack VALUES (1, 'framework', 'React', '18.2', 'package.json', datetime('now'))");
      
      const model = new PersistentProjectModelImpl(db);
      await model.loadFromDb();
      
      expect(model.getTechStack()).toContainEqual(
        expect.objectContaining({ name: 'React', version: '18.2' })
      );
    });
    
    it('handles corrupted database gracefully', async () => {
      const model = new PersistentProjectModelImpl(corruptedDb);
      await expect(model.loadFromDb()).rejects.toThrow();
      // Model should remain empty, not crash
    });
    
    it('loads empty database (first run after Phase 1)', async () => {
      const db = createEmptyDb();
      const model = new PersistentProjectModelImpl(db);
      await model.loadFromDb();
      
      expect(model.isPopulated()).toBe(false);
      expect(model.getTechStack()).toEqual([]);
    });
  });
  
  describe('reconcileWithFileSystem', () => {
    it('detects changed package.json', async () => {
      const model = await createPopulatedModel();
      // Modify package.json on disk
      await addDependency('new-package', '1.0.0');
      
      const result = await model.reconcileWithFileSystem();
      
      expect(result.status).toBe('reconciled');
      expect(result.categoriesUpdated).toContain('dependencies');
    });
    
    it('returns in-sync when nothing changed', async () => {
      const model = await createPopulatedModel();
      const result = await model.reconcileWithFileSystem();
      
      expect(result.status).toBe('in-sync');
      expect(result.durationMs).toBeLessThan(1000);
    });
    
    it('full rebuild when >50% categories changed', async () => {
      const model = await createPopulatedModel();
      // Change deps + config + structure
      await modifyMultipleFiles();
      
      const result = await model.reconcileWithFileSystem();
      expect(result.status).toBe('rebuilt');
    });
  });
  
  describe('incremental updates', () => {
    it('updates tech stack on DEPENDENCY_CHANGE', async () => {
      const model = await createPopulatedModel();
      await model.applyFileChange({
        filePath: 'package.json',
        eventType: 'change',
        classifiedAs: 'DEPENDENCY_CHANGE',
        timestamp: new Date()
      });
      
      // Verify model updated
      expect(model.getTechStack().length).toBeGreaterThan(0);
    });
    
    it('emits modelChanged event on update', async () => {
      const model = await createPopulatedModel();
      const listener = vi.fn();
      model.onModelChanged(listener);
      
      await model.applyFileChange(dependencyChange);
      
      expect(listener).toHaveBeenCalledWith(
        expect.objectContaining({ techStack: expect.any(Array) })
      );
    });
  });
  
  describe('debounced writes', () => {
    it('batches writes within 5 seconds', async () => {
      const model = await createPopulatedModel();
      const saveSpy = vi.spyOn(model, 'saveToDb');
      
      await model.applyFileChange(change1);
      await model.applyFileChange(change2);
      await model.applyFileChange(change3);
      
      // Should NOT have saved yet
      expect(saveSpy).not.toHaveBeenCalled();
      
      // Wait for debounce
      await vi.advanceTimersByTime(5000);
      
      // Should have saved exactly once
      expect(saveSpy).toHaveBeenCalledTimes(1);
    });
    
    it('flushes on deactivation', async () => {
      const model = await createPopulatedModel();
      await model.applyFileChange(change1);
      
      await model.deactivate();
      
      // Verify data is in SQLite
      const rows = model.database.prepare('SELECT * FROM tech_stack').all();
      expect(rows.length).toBeGreaterThan(0);
    });
  });
});
```

---

## Performance Budgets

| Operation | Budget | Notes |
| --- | --- | --- |
| loadFromDb() | <500ms | Typical project: 50-100 rows total |
| saveToDb() | <200ms | Transaction batches all writes |
| reconcileWithFileSystem() | <1s | Compares file timestamps, not full content |
| applyFileChange() | <100ms | Single file parse + model update |
| isPopulated() | <1ms | Boolean check on in-memory state |

---

## Build Prompt for AI Agent

```
Build the Project Model Persistence module according to this spec.

This module EXTENDS (does not modify) Phase 1's ProjectModel.
Use the PersistentProjectModel interface that extends ProjectModel.

Key requirements:
1. Load model from SQLite at activation (<500ms)
2. Validate against current file system (reconciliation)
3. Apply incremental updates from file watcher events
4. Debounced writes to SQLite (5 second batches)
5. Emit modelChanged events for generators to subscribe to
6. isPhase15Active() guard for conditional activation
7. Handle corrupted database (delete and rebuild)
8. Flush on deactivation (never lose data)
9. All file I/O is async (no blocking operations)
10. Include 20+ unit tests

Files to create:
- src/model/project-model-persistence.ts (~250 lines)
- src/model/project-model-persistence.test.ts (~300 lines)
- Updated src/types.ts (add PersistentProjectModel interface)

Verification:
- npm run test passes
- npm run lint passes
- Phase 1 tests still pass (backward compatible)
```

---

**CRITICAL DEPENDENCY:** This module must be built FIRST in Phase 1.5. The File Watcher, all Generators, Edit Tracker, and Learning Database depend on it.