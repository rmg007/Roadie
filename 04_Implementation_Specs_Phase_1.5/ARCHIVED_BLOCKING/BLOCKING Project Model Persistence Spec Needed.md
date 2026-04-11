# 🔴 BLOCKING: Project Model Persistence Spec Needed (M14)

**Priority:** BLOCKING — Everything else depends on this  

**Blocking:** File generators, Edit Tracker, Learning Database  

**Time to Build:** 4-5 hours  

**Scope:** This is a NEW specification that doesn't exist yet

---

## Why This Is Blocking

The build order for Phase 1.5 is:

```
M14: Project Model Persistence (M14)
      ↓ (everything depends on this)
  ├─ M15: File Watcher
  ├─ M16: Project Model Updater
  ├─ M19: File Generator Manager
  ├─ M20-27: File-specific generators
  ├─ M21: Edit Tracker
  └─ M23: Learning Database
```

Without M14 spec, cannot write specs for:

- M15 (File Watcher) — needs to know how to update project model
- M19-27 (Generators) — need to know model's query APIs
- M21 (Edit Tracker) — needs to know model's schema
- M23 (Learning DB) — needs to know shared database with model

---

## What M14 Needs to Specify

### 1. SQLite Schema (Extending Phase 1 In-Memory Model)

**Phase 1 had these in memory:**

- Tech stack (language, frameworks, versions)
- Directory structure (source, test, config roots)
- Package manager, test runner, linter, formatter
- Commands (build, test, dev, lint, format)

**Phase 1.5 must persist these to SQLite:**

```sql
CREATE TABLE tech_stack (
  id INTEGER PRIMARY KEY,
  category TEXT, -- 'language', 'framework', 'build_tool', etc.
  name TEXT,
  version TEXT,
  source_file TEXT,
  detected_at TEXT
);

CREATE TABLE directory_structure (
  id INTEGER PRIMARY KEY,
  path TEXT UNIQUE,
  type TEXT, -- 'directory' or 'file'
  role TEXT, -- 'source', 'test', 'config', 'output', 'static'
  language TEXT,
  last_scanned TEXT
);

CREATE TABLE commands (
  id INTEGER PRIMARY KEY,
  name TEXT,
  command TEXT,
  source_file TEXT,
  type TEXT, -- 'build', 'test', 'dev', 'lint', 'format', 'other'
  last_used TEXT
);

-- ... more tables for dependencies, patterns, etc.
```

### 2. Load/Save/Validate Operations

**Load at Activation:**

```
async function loadProjectModel() {
  if (.github/.roadie/project-model.db exists) {
    db = open(.github/.roadie/project-model.db)
    model = ProjectModel.fromDB(db)
    return model
  } else {
    return ProjectModel.empty()
  }
end
```

**Save on Shutdown:**

```
async function saveProjectModel(model) {
  db = open(.github/.roadie/project-model.db)
  db.transaction(() => {
    for each table:
      delete old data
      insert new data
  })
  db.close()
end
```

**Validate at Startup:**

```
async function validateModel(model) {
  // Compare loaded model with current file system
  // Check if tech stack matches current package.json
  // Check if source structure matches current fs
  // Log warnings if out of sync
  
  if (out of sync):
    reconcile() // update model
end
```

### 3. Incremental Updates from File Watcher

When File Watcher detects a change:

```
async function updateModel(changeType, filePath) {
  if changeType == 'DEPENDENCY_CHANGE':
    scanDependencies() // update tech_stack table
    
  if changeType == 'CONFIG_CHANGE':
    scanConfig(filePath) // update commands/config table
    
  if changeType == 'STRUCTURE_CHANGE':
    scanStructure() // update directory_structure table
    
  // Debounced write to SQLite (batch every 5 seconds)
  debouncedSave()
end
```

### 4. Query APIs for Generators

Generators need to query the model:

```tsx
interface PersistentProjectModel {
  // Tech stack queries
  getTechStack(): TechStackEntry[];
  getFramework(): Framework | null;
  getLanguage(): Language | null;
  
  // Structure queries
  getSourceStructure(path?: string): DirectoryNode;
  getSourceDirectories(): string[];
  
  // Command queries
  getCommand(type: 'test' | 'build' | 'lint'): string | null;
  
  // Dependency queries
  getDependencies(): Map<string, string>;
  getDevDependencies(): Map<string, string>;
  
  // Persistence
  async saveToDb(): Promise<void>;
  async loadFromDb(): Promise<void>;
  async reconcileWithFileSystem(): Promise<void>;
}
```

### 5. Migration from Phase 1

Phase 1 had in-memory model only. Phase 1.5 must:

- Load from memory if DB doesn't exist
- Detect this is first run (create DB)
- Persist the model
- Next activation loads from DB

```
async function bootstrap() {
  if (no DB && Phase 1 model exists in memory) {
    // First time activating Phase 1.5
    model = loadPhase1Model()
    await model.saveToDb()
  }
end
```

---

## Sections That Should Be in the Spec

1. **Module Identity**
    - M14: Project Model Persistence
    - Depends on: types.ts
    - Used by: All other Phase 1.5 modules
2. **Responsibility**
    - Load project model from SQLite at activation
    - Persist incremental updates
    - Validate against file system
    - Reconcile if out of sync
    - Expose query APIs for generators
3. **SQLite Schema**
    - Complete CREATE TABLE statements
    - Indexes for performance
    - Schema versioning strategy
    - Migration path
4. **Interface & Public API**
    - Methods for load/save/validate
    - Query methods for generators
    - Error handling
5. **Startup/Shutdown Sequence**
    - What happens on activation
    - What happens on deactivation
    - Crash recovery
6. **Testing Strategy**
    - Unit tests (DB operations)
    - Integration tests (real SQLite)
    - Migration tests (Phase 1 → Phase 1.5)
7. **Performance Budgets**
    - Load time < 500ms
    - Query time < 100ms
    - Write time < 200ms

---

## Database Design Decisions

### Single DB vs Two DBs

**Decision: Single unified database**

Contains:

- Project model tables (tech_stack, directory_structure, commands)
- Learning tables (edit_history, workflow_outcomes, patterns)
- Both in single `.github/.roadie/project-model.db`

**Why:**

- Simpler transaction management
- One connection to manage
- One file to backup/restore
- Easier to reason about consistency

### Persistence Strategy

**Debounced writes (every 5 seconds max)**

Not persisting on every change (expensive), but not waiting indefinitely (data loss on crash).

### Validation Strategy

**Compare mod times**

- Load model
- Check `package.json` mod time > model's last_analyzed
- If newer: re-scan dependencies
- Same for tsconfig.json, src/, etc.

---

## Build Prompt for AI Agent

```
Build the Project Model Persistence module (M14) specification.

This is the FIRST module to build (blocker for all others).

Key requirements:
1. SQLite schema extending Phase 1 in-memory model
2. Load/save/validate operations
3. Incremental updates from file watcher
4. Query APIs for generators (getTechStack, getSourceStructure, etc.)
5. Migration from Phase 1 (in-memory to persisted)
6. Performance budgets: load < 500ms, query < 100ms
7. Crash recovery and data integrity
8. Unit tests (DB operations) + integration tests (real SQLite)

Files to create:
- src/model/project-model-persistence.ts (main module)
- src/model/database-schema.ts (SQL schema)
- src/model/database-operations.ts (CRUD operations)
- test/model/project-model-persistence.test.ts (tests)
- test/integration/project-model-persistence.integration.test.ts

Verification criteria:
- npm run test passes
- npm run lint passes
- Load from SQLite: < 500ms
- Query performance: < 100ms
- Phase 1 migration test passes
- Startup reconciliation test passes
```

---

## Blocking This

M14 is the absolute blocker. Once this spec exists and is clear, all other Phase 1.5 specs can be written.

**Next:** FW-1 (File Watcher API restructuring) and SM-1 (Section Manager merge) are also blocking, but M14 must be first.

---

**Status:** NEEDED (not yet created)  

**Owner:** AI Agent (to be assigned)  

**Estimated Time:** 4-5 hours to create spec