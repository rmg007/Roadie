# 👁️ File Watcher Manager Specification

## Monitors Workspace for Changes, Classifies Events, Dispatches to Updaters

---

## Module Identity

**Module ID:** M15  

**File Location:** `src/watcher/file-watcher-manager.ts`  

**Depends On:** Project Model Persistence (M16), Configuration (Phase 1)  

**Used By:** Project Model Updater, extension activation/deactivation  

**Complexity:** Medium (edge cases in debouncing, classification, error handling)  

**Estimated Build Time:** 4-5 hours  

---

## Responsibility

Detect file system changes and notify the project model updater, which triggers generators. This is the "heartbeat" of passive mode.

**Three critical jobs:**

1. **Watch Files:** Monitor for changes using VS Code FileSystemWatcher
2. **Classify Events:** Determine what type of change occurred
3. **Dispatch:** Route events to appropriate updaters

---

## Watched Paths & Glob Patterns

### Watched (Active)

```tsx
// Dependency files (highest priority)
"package.json"
"package-lock.json"
"yarn.lock"
"pnpm-lock.yaml"
"Gemfile" / "Gemfile.lock"      // Ruby
"requirements.txt" / "poetry.lock" // Python
"go.mod" / "go.sum"             // Go
"Cargo.toml" / "Cargo.lock"     // Rust
"composer.json" / "composer.lock" // PHP

// Configuration files
"tsconfig.json"
"tsconfig.*.json"
"jest.config.*"
"vitest.config.*"
"eslint.config.*"
".babelrc*"
"vite.config.*"
"webpack.config.*"

// Source code structure (watch for new directories)
"src/"       // All changes
"tests/" or "__tests__/" // All changes
"components/" // etc.

// GitHub config (watch for human edits)
".github/copilot-*.md"
".github/agents/*.yaml"
".github/skills/*.md"
".github/workflows/*.yml"
// (but NOT .github/.roadie/ — that's Roadie's private data)
```

### Explicitly Ignored (Never watched)

```tsx
// Version control
".git/**"
".gitignore"
".gitattributes"

// Dependencies
"node_modules/**"
"vendor/**"
"venv/**"
".venv/**"
"site-packages/**"

// Build artifacts
"dist/**"
"build/**"
"out/**"
"target/**"
"bin/**"
"obj/**"

// Cache
".next/**"
".cache/**"
".eslintcache"
".parcel-cache/**"

// IDE
".vscode/**"
".idea/**"
"*.swp"
"*.swo"

// OS
".DS_Store"
"Thumbs.db"

// Logs
"*.log"
"logs/**"

// Temp
"tmp/**"
"temp/**"
```

---

## Change Classification

### Classification Algorithm

```jsx
function classifyChange(filePath, eventType) {
  // eventType: 'add' | 'change' | 'unlink' | 'addDir' | 'unlinkDir'
  
  // 1. Check if it's a dependency file
  if (isDependencyFile(filePath)) {
    return {
      type: 'DEPENDENCY_CHANGE',
      priority: 'HIGH',
      triggers: ['copilot-instructions', 'agents', 'skills']
    };
  }
  
  // 2. Check if it's a config file
  if (isConfigFile(filePath)) {
    return {
      type: 'CONFIG_CHANGE',
      priority: 'MEDIUM',
      triggers: ['path-instructions', 'workflows']
    };
  }
  
  // 3. Check if it's a source structure change (inferred from file path)
  if (isNewDirectoryInferred(filePath, projectModel)) {
    return {
      type: 'STRUCTURE_CHANGE',
      priority: 'MEDIUM',
      triggers: ['path-instructions']
    };
  }
  
  // 4. Check if it's a new source file
  if (isSourceFile(filePath) && eventType === 'add') {
    return {
      type: 'SOURCE_ADDITION',
      priority: 'LOW',
      triggers: [] // No generators triggered
    };
  }
  
  // 5. Check if it's a user edit to generated file
  if (isGeneratedFile(filePath) && eventType === 'change') {
    return {
      type: 'USER_EDIT',
      priority: 'MEDIUM',
      triggers: ['edit-tracker']
    };
  }
  
  // 6. Unknown
  return {
    type: 'OTHER',
    priority: 'LOW',
    triggers: []
  };
end
```

### Change Types

| Type | Triggers | Example | Priority |
| --- | --- | --- | --- |
| `DEPENDENCY_CHANGE` | Copilot Instructions, Agents | package.json updated | HIGH |
| `CONFIG_CHANGE` | Path Instructions, Workflows | tsconfig.json changed | MEDIUM |
| `STRUCTURE_CHANGE` | Path Instructions | src/components/ added | MEDIUM |
| `SOURCE_ADDITION` | (none) | src/index.ts created | LOW |
| `USER_EDIT` | Edit Tracker | .github/copilot-*.md modified | MEDIUM |
| `OTHER` | (none) | Random file in project | LOW |

---

## Debouncing & Batching

### Why Debounce?

File system events often come in bursts (e.g., git checkout, npm install). Without debouncing:

- 100+ events in 1 second
- Each triggers project model update
- Each triggers generators
- Performance nightmare

### Debounce Strategy

```
function setupDebouncing() {
  const pendingEvents = [];
  const debounceTimer = 500ms; // Configurable: roadie.fileWatcherTimeout
  
  return function onFileChange(filePath, eventType) {
    // Add to pending
    pendingEvents.push({filePath, eventType, timestamp: now});
    
    // Reset timer
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
      // All events collected, process them
      processCollectedEvents(pendingEvents);
      pendingEvents.clear();
    }, 500ms);
  };
end

function processCollectedEvents(events) {
  // 1. Deduplicate
  const unique = deduplicateEvents(events);
  // If same file changed 3x quickly, only process once
  
  // 2. Classify each
  const classified = unique.map(e => classifyChange(e.filePath, e.eventType));
  
  // 3. Batch by type
  const byType = groupBy(classified, 'type');
  
  // 4. Dispatch
  for (const [changeType, events] of byType) {
    dispatcher.dispatch(changeType, events);
  }
end
```

### Event Deduplication

```
function deduplicateEvents(events) {
  const seen = new Map<string, Event>();
  
  // Process in order, keep LAST occurrence of each file
  for (const event of events) {
    seen.set(event.filePath, event);
  }
  
  // If last event was 'unlink', and earlier was 'add' for same file,
  // remove both (file was created and deleted within debounce window)
  const toRemove = new Set();
  for (const [path, event] of seen) {
    if (event.eventType === 'unlink') {
      // Check if 'add' exists for same file
      const earlierAdd = events.find(e => e.filePath === path && e.eventType === 'add');
      if (earlierAdd) {
        toRemove.add(path);
      }
    }
  }
  
  // Remove cancelled-out events
  for (const path of toRemove) {
    seen.delete(path);
  }
  
  return Array.from(seen.values());
end
```

---

## Dispatcher Logic

### Dispatch to Project Model Updater

```
function dispatch(changeType, events) {
  // Route to appropriate handler
  
  switch (changeType) {
    case 'DEPENDENCY_CHANGE':
      projectModelUpdater.updateDependencies(events);
      break;
    
    case 'CONFIG_CHANGE':
      projectModelUpdater.updateConfiguration(events);
      break;
    
    case 'STRUCTURE_CHANGE':
      projectModelUpdater.updateSourceStructure(events);
      break;
    
    case 'USER_EDIT':
      editTracker.recordEdit(events);
      break;
    
    case 'SOURCE_ADDITION':
      // No action needed (just for logging)
      logger.debug(`Source file added: ${events.map(e => e.filePath).join(', ')}`);
      break;
    
    case 'OTHER':
      // Ignore
      break;
  }
end
```

---

## Error Handling

### Scenario 1: Permission Denied

```
watcher.on('error', (error) => {
  if (error.code === 'EACCES') { // Permission denied
    logger.warn(`Permission denied watching ${error.path}`);
    
    // Skip this path, continue watching others
    // Don't crash the whole watcher
    skipPath(error.path);
  } else {
    // Other error
    logger.error(`Watcher error: ${error.message}`);
    
    // Try to recover
    attemptRecovery();
  }
});
```

### Scenario 2: Watcher Crashes / Becomes Unresponsive

```
function setupWatcherHealthCheck() {
  const lastEventTime = now();
  const healthCheckInterval = 30 seconds; // configurable
  
  const timer = setInterval(() => {
    if (now() - lastEventTime > 2 * healthCheckInterval) {
      // No events in 2x interval
      // Watcher is likely dead
      logger.warn("File watcher appears unresponsive, restarting...");
      
      restartWatcher(); // Close old, start new
      
      // Trigger full rescan (force project model rebuild)
      projectModelUpdater.rescanEverything();
    }
  }, healthCheckInterval);
  
  watcher.on('all', () => {
    lastEventTime = now();
  });
end
```

### Scenario 3: Large Batch of Changes (e.g., git checkout)

```
function processCollectedEvents(events) {
  // If more than 1000 events in one batch
  if (events.length > 1000) {
    logger.info(`Large batch detected (${events.length} events), triggering full rescan`);
    
    // Don't try to process individually
    // Just trigger a full rebuild
    projectModelUpdater.rescanEverything();
    return;
  }
  
  // Normal processing...
end
```

### Scenario 4: Out of Memory / Watcher Limits

```
if (watchedPathCount > config.maxWatchedPaths) {
  logger.warn(`Exceeded max watched paths (${watchedPathCount}), switching to polling`);
  
  // Close native watchers
  watcher.close();
  
  // Switch to polling strategy
  startPollingWatcher();
}
```

---

## Polling Fallback

For large workspaces or when native watchers fail:

```jsx
function startPollingWatcher() {
  const pollInterval = 5000ms; // 5 seconds, configurable
  const lastState = new Map<string, {mtime, size}>();
  
  // IMPORTANT: Only poll WATCHED directories (dependency files, config files)
  // Do NOT poll the entire workspace. Source directories use a longer interval.
  const watchedPatterns = getDependencyAndConfigGlobs(); // package.json, tsconfig, etc.
  const sourcePatterns = getSourceDirectoryGlobs();       // src/, components/, etc.
  const sourcePollInterval = 30000; // 30 seconds for source structure
  
  // Poll dependency/config files frequently
  setInterval(() => {
    const currentState = scanMatchingFiles(watchedPatterns);
    processChanges(lastState, currentState);
  }, pollInterval);
  
  // Poll source structure less frequently (new directories only)
  setInterval(() => {
    const currentDirs = scanDirectories(sourcePatterns);
    processStructureChanges(lastDirState, currentDirs);
  }, sourcePollInterval);
    
    // Compute diff
    const changes = [];
    for (const [path, stat] of currentState) {
      if (!lastState.has(path)) {
        changes.push({path, eventType: 'add', stat});
      } else {
        const oldStat = lastState.get(path);
        if (oldStat.mtime !== stat.mtime || oldStat.size !== stat.size) {
          changes.push({path, eventType: 'change', stat});
        }
      }
    }
    
    // Check for deletions
    for (const [path, stat] of lastState) {
      if (!currentState.has(path)) {
        changes.push({path, eventType: 'unlink'});
      }
    }
    
    if (changes.length > 0) {
      // Process like normal watcher events
      onBatchChanges(changes);
    }
    
    lastState = currentState;
  }, pollInterval);
end
```

**Performance note:** Polling is slower but reliable. Use native watchers when possible.

---

## Event Types & Handling

### VS Code FileSystemWatcher Events

> **Important:** VS Code FileSystemWatcher provides `onDidCreate`, `onDidChange`, and `onDidDelete` events for files only. It does NOT provide directory-specific events (`addDir`/`unlinkDir`). Directory changes are inferred from file creation/deletion paths. Use `vscode.workspace.createFileSystemWatcher(globPattern)` to create watchers.
> 

```tsx
const watcher = vscode.workspace.createFileSystemWatcher('**/package.json');

watcher.onDidCreate((uri) => {
  emit('file-change', {filePath: uri.fsPath, eventType: 'create'});
});

watcher.onDidChange((uri) => {
  emit('file-change', {filePath: uri.fsPath, eventType: 'change'});
});

watcher.onDidDelete((uri) => {
  emit('file-change', {filePath: uri.fsPath, eventType: 'delete'});
});

// Register multiple watchers for different patterns
const watchers = [
  vscode.workspace.createFileSystemWatcher('**/package.json'),
  vscode.workspace.createFileSystemWatcher('**/tsconfig.json'),
  vscode.workspace.createFileSystemWatcher('**/.eslintrc*'),
  vscode.workspace.createFileSystemWatcher('**/vitest.config*'),
  vscode.workspace.createFileSystemWatcher('.github/**/*.md'),
];

// Dispose all on deactivation
context.subscriptions.push(...watchers);
```

> **Note on directory events:** When a file is created in a new directory (e.g., `src/components/Button.tsx`), infer `STRUCTURE_CHANGE` from the path if the parent directory didn't previously exist in the project model. When all files in a directory are deleted, infer directory removal from the project model's directory tree.
> 

---

## Interface & Public API

```tsx
interface FileWatcherManager {
  // Activation/deactivation
  start(config: FileWatcherConfig): Promise<void>;
  stop(): Promise<void>;
  
  // Event subscription
  on(event: 'file-change', handler: (change: FileChange) => void): void;
  on(event: 'error', handler: (error: Error) => void): void;
  
  // Commands
  rescan(): Promise<void>; // Force full rescan
  ignorePattern(pattern: string): void; // Dynamically ignore
  
  // Status
  isWatching(): boolean;
  getWatchedPathCount(): number;
  getStatus(): WatcherStatus;
}

interface FileChange {
  filePath: string;
  eventType: 'add' | 'change' | 'unlink' | 'addDir' | 'unlinkDir';
  timestamp: Date;
  classifiedAs: ChangeType;
}

type ChangeType = 
  | 'DEPENDENCY_CHANGE'
  | 'CONFIG_CHANGE'
  | 'STRUCTURE_CHANGE'
  | 'SOURCE_ADDITION'
  | 'USER_EDIT'
  | 'OTHER';

interface FileWatcherConfig {
  workspace: string; // VS Code workspace path
  debounceMs: number; // default: 500
  maxWatchedPaths: number; // default: 5000
  usePolling: boolean; // default: false
  pollIntervalMs: number; // default: 5000
}

interface WatcherStatus {
  watching: boolean;
  mode: 'native' | 'polling';
  watchedPaths: number;
  totalEventsSinceStart: number;
  lastEventTime: Date | null;
  errors: Error[];
}
```

---

## Testing Strategy

### Unit Tests

```tsx
// test/watcher/classification.test.ts
describe('Change Classification', () => {
  it('classifies package.json change as DEPENDENCY_CHANGE', () => {
    const change = classifyChange('package.json', 'change');
    expect(change.type).toBe('DEPENDENCY_CHANGE');
    expect(change.priority).toBe('HIGH');
  });
  
  it('classifies src/components/ addition as STRUCTURE_CHANGE', () => {
    const change = classifyChange('src/components', 'addDir');
    expect(change.type).toBe('STRUCTURE_CHANGE');
  });
  
  it('classifies .github/copilot.md change as USER_EDIT', () => {
    const change = classifyChange('.github/copilot-instructions.md', 'change');
    expect(change.type).toBe('USER_EDIT');
  });
  // ... 20+ more classification tests
});

// test/watcher/debounce.test.ts
describe('Debouncing', () => {
  it('batches rapid changes to same file', async () => {
    const received: FileChange[] = [];
    const watcher = new MockWatcher();
    
    watcher.on('batch', (changes) => {
      received.push(...changes);
    });
    
    // Emit 3 rapid changes to same file
    watcher.emit('change', 'package.json');
    watcher.emit('change', 'package.json');
    watcher.emit('change', 'package.json');
    
    // Wait for debounce
    await wait(600);
    
    // Should receive 1 (deduplicated)
    expect(received.length).toBe(1);
  });
  
  it('batches many files within debounce window', async () => {
    const batches: FileChange[][] = [];
    const watcher = new MockWatcher();
    
    watcher.on('batch', (changes) => {
      batches.push(changes);
    });
    
    // Emit 100 changes within 500ms
    for (let i = 0; i < 100; i++) {
      watcher.emit('change', `src/file${i}.ts`);
    }
    
    await wait(600);
    
    // Should receive 1 batch (not 100)
    expect(batches.length).toBe(1);
    expect(batches[0].length).toBe(100);
  });
  // ... more debounce tests
});

// test/watcher/error-handling.test.ts
describe('Error Handling', () => {
  it('recovers from permission denied errors', async () => {
    const watcher = new MockWatcher();
    const errors: Error[] = [];
    
    watcher.on('error', (err) => {
      errors.push(err);
    });
    
    // Simulate permission error
    watcher.emit('error', new Error('EACCES'));
    
    // Watcher should still be running
    expect(watcher.isWatching()).toBe(true);
  });
  
  it('restarts watcher if unresponsive', async () => {
    const watcher = new MockWatcher();
    let restartCount = 0;
    
    watcher.on('restart', () => {
      restartCount++;
    });
    
    // Simulate 2 minute silence
    watcher.simulateNoEventsFor(120000);
    
    // Should trigger restart
    expect(restartCount).toBeGreaterThan(0);
  });
});
```

### Integration Tests

```tsx
// test/integration/file-watcher.integration.test.ts
describe('File Watcher Integration', () => {
  let tempDir: string;
  let watcher: FileWatcherManager;
  
  beforeEach(() => {
    tempDir = fs.mkdtempSync();
    watcher = new FileWatcherManager();
  });
  
  afterEach(() => {
    watcher.stop();
    fs.rmSync(tempDir, {recursive: true});
  });
  
  it('detects dependency file changes', async () => {
    const changes: FileChange[] = [];
    watcher.on('file-change', (change) => {
      changes.push(change);
    });
    
    await watcher.start({workspace: tempDir});
    
    // Modify package.json
    const packagePath = path.join(tempDir, 'package.json');
    fs.writeFileSync(packagePath, '{"name": "test"}')
    
    // Create it first
    fs.writeFileSync(packagePath, '{"name": "old"}');
    
    // Now change it
    await wait(100);
    fs.writeFileSync(packagePath, '{"name": "new"}');
    
    // Wait for debounce
    await wait(600);
    
    // Should detect as DEPENDENCY_CHANGE
    expect(changes.some(c => c.classifiedAs === 'DEPENDENCY_CHANGE')).toBe(true);
  });
  
  it('ignores node_modules changes', async () => {
    const changes: FileChange[] = [];
    watcher.on('file-change', (change) => {
      changes.push(change);
    });
    
    await watcher.start({workspace: tempDir});
    
    // Create node_modules directory and file
    const nmPath = path.join(tempDir, 'node_modules', 'package', 'index.js');
    fs.mkdirSync(path.dirname(nmPath), {recursive: true});
    fs.writeFileSync(nmPath, 'module.exports = {}');
    
    await wait(600);
    
    // Should NOT detect
    expect(changes.length).toBe(0);
  });
});
```

---

## Performance Budgets

| Operation | Budget | Notes |
| --- | --- | --- |
| Classify change | < 5ms | Per event |
| Debounce + batch | < 500ms | Configurable |
| Process batch | < 500ms | 1000 events |
| Dispatch to updater | < 10ms | Async |
| Total event latency | < 1.5s | Event → classify (5ms) + debounce (500ms) + process (200ms) + dispatch (10ms) = ~715ms typical |

---

## Configuration Options

```tsx
// In package.json contributes.configuration
{
  "roadie.fileWatcherTimeout": {
    "type": "number",
    "default": 500,
    "description": "Debounce time for file watcher events (ms)"
  },
  "roadie.maxWatchedPaths": {
    "type": "number",
    "default": 5000,
    "description": "Maximum paths to watch before switching to polling"
  },
  "roadie.fileWatcherUsePolling": {
    "type": "boolean",
    "default": false,
    "description": "Force polling mode (slower, more compatible)"
  }
}
```

---

## Build Prompt for AI Agent

```jsx
Build the File Watcher Manager module (M15) according to this spec.

Key requirements:
1. Watch files using VS Code FileSystemWatcher with 500ms custom debounce
2. Classify changes (DEPENDENCY, CONFIG, STRUCTURE, etc.)
3. Infer directory changes from file creation/deletion paths (VS Code watcher has no addDir/unlinkDir events)
4. Gracefully handle errors (permission denied, watcher crash)
5. Fallback to polling if native watchers fail
6. Emit events to dispatcher
7. On startup, reconcile watched files against project model timestamps
8. Include 30+ unit tests
9. Include 5+ integration tests

Files to create:
- src/watcher/file-watcher-manager.ts (main module, ~250 lines)
- src/watcher/change-classifier.ts (classification logic, ~100 lines)
- src/watcher/event-dispatcher.ts (routing, ~80 lines)
- test/watcher/*.test.ts (30+ tests)

Verification criteria:
- npm run test passes
- npm run lint passes
- All classification tests pass
- Integration test with real file system passes
```

---

**Next Module:** M16 - Project Model Persistence (extends Phase 1 project model with SQLite)

**Critical Dependency:** This module must work before building generators (M20+)