# File Watcher Manager Specification

## Monitors Workspace for Changes, Classifies Events, Triggers Auto-Rescan

---

## Module Identity

**Module ID:** M15

**Files:**
- `src/watcher/file-watcher-manager.ts` — debouncing, deduplication, batched dispatch
- `src/watcher/change-classifier.ts` — pure classification logic (no VS Code dependency)

**Depends On:** `change-classifier`

**Used By:** `extension.ts` (activation wiring)

**Implementation Status:** ✅ COMPLETE and CONNECTED — Wired into `extension.ts` as of 2026-04-14

---

## Responsibility

When a dependency file (`package.json`, lock files) or config file (`tsconfig.json`, `vite.config.*`, etc.) changes on disk, Roadie automatically:

1. Detects the change via VS Code `FileSystemWatcher`
2. Debounces and deduplicates the raw events (500 ms window)
3. Classifies each event by type and priority
4. Triggers a full project re-analysis (`ProjectAnalyzer.analyze()`)
5. Regenerates `.github/` files if the analysis produces new content

**This means Roadie stays in sync automatically.** If you switch from npm to pnpm (add `pnpm-lock.yaml`), Roadie detects the new lock file, re-runs analysis, updates all stored commands from `npm run …` to `pnpm run …`, and rewrites `.github/copilot-instructions.md` — without any manual action.

---

## Architecture

```
VS Code FileSystemWatcher (glob pattern)
    │  onDidCreate / onDidChange / onDidDelete
    ▼
FileWatcherManager.handleFileEvent(filePath, type)
    │  isIgnoredPath() → drop silently
    │  deduplication: keep latest event per path
    │  add+delete cancellation within window
    │  500 ms debounce timer reset on each event
    ▼
FileWatcherManager.processBatch()
    │  > 1000 events → emit FullRescanEvent sentinel
    │  else → classifyChange() for each pending event
    │  sort HIGH → MEDIUM → LOW
    ▼
BatchHandler (extension.ts)
    │  FullRescanEvent → runRescan()
    │  any HIGH (DEPENDENCY_CHANGE) or MEDIUM (CONFIG_CHANGE) → runRescan()
    │  LOW only → skip
    ▼
runRescan()
    ├─ ProjectAnalyzer.analyze(workspaceRoot)
    └─ FileGenerator.generateAll(projectModel) → write .github/ files
```

---

## Wired-Up Glob Pattern (extension.ts)

A single `vscode.workspace.createFileSystemWatcher` covers all relevant files:

```
**/{package.json,package-lock.json,pnpm-lock.yaml,yarn.lock,bun.lockb,
    tsconfig.json,tsconfig.*.json,jest.config.*,vitest.config.*,
    vite.config.*,webpack.config.*,rollup.config.*,.babelrc,.eslintrc*,.prettierrc*}
```

VS Code's `FileSystemWatcher` uses the workspace root as the base, so only files inside the open workspace folder are watched. Node modules and other large trees are not traversed because the glob matches specific filenames only.

---

## Change Classification

### `classifyChange(filePath, eventType)` → `ClassifiedChange`

Classification runs in strict priority order. First match wins.

| Priority | Classified as | Condition |
|---|---|---|
| 1 | `DEPENDENCY_CHANGE` | Basename is in `DEPENDENCY_FILES` set |
| 2 | `CONFIG_CHANGE` | Basename matches a `CONFIG_PATTERNS` regex |
| 3 | `USER_EDIT` | Path contains `.github/copilot-`, `.github/agents/`, or `.github/skills/` |
| 4 | `SOURCE_ADDITION` | `eventType === 'create'` and extension is `.ts`, `.js`, `.tsx`, `.jsx`, `.py`, `.go`, or `.rs` |
| 5 | `OTHER` | Everything else |

### DEPENDENCY_FILES (exact basename match)

All node ecosystems:
```
package.json   package-lock.json   yarn.lock   pnpm-lock.yaml   bun.lockb
```
Go: `go.mod`, `go.sum`
Rust: `Cargo.toml`, `Cargo.lock`
Python: `requirements.txt`, `Pipfile`, `Pipfile.lock`, `poetry.lock`, `pyproject.toml`
PHP: `composer.json`, `composer.lock`
Ruby: `Gemfile`, `Gemfile.lock`

> `bun.lockb` was added to align with the package manager detector in `dependency-scanner.ts`, which checks for `bun.lockb` to select the `bun` package manager.

### CONFIG_PATTERNS (regex on basename)

```
/^tsconfig(\..+)?\.json$/    tsconfig.json, tsconfig.app.json, etc.
/^jest\.config\..+$/
/^vitest\.config\..+$/
/^eslint\.config\..+$/
/^webpack\.config\..+$/
/^vite\.config\..+$/
/^\.babelrc/
/^\.eslintrc/
/^\.prettierrc/
/^rollup\.config\..+$/
```

### Priority mapping

| `classifiedAs` | `priority` | Triggers re-analysis? |
|---|---|---|
| `DEPENDENCY_CHANGE` | `HIGH` | ✅ Yes |
| `CONFIG_CHANGE` | `MEDIUM` | ✅ Yes |
| `USER_EDIT` | `MEDIUM` | ✅ Yes (via HIGH/MEDIUM filter) |
| `SOURCE_ADDITION` | `LOW` | ❌ No |
| `OTHER` | `LOW` | ❌ No |

---

## Ignored Paths

`isIgnoredPath()` normalizes backslashes then rejects any path that contains or starts with one of these prefixes:

```
node_modules/    .git/    dist/    build/    out/
.next/           .cache/  .vscode/ .idea/
vendor/          venv/    .venv/
```

Events for ignored paths are dropped immediately inside `handleFileEvent()` — they never enter the pending map or the debounce timer.

---

## Debouncing and Deduplication

`FileWatcherManager` uses a `Map<string, PendingEvent>` (keyed by file path) with a rolling 500 ms debounce timer.

**Deduplication:** When the same file fires multiple events within the debounce window, only the last one is kept.

**Add+delete cancellation:** If a `create` event is pending for a path and a `delete` arrives (or vice-versa), both are removed. The net effect of creating then deleting within 500 ms is nothing — no re-analysis is triggered.

**Batch overflow:** If more than 1000 events accumulate before the debounce fires (e.g., a large `git checkout`), a `FullRescanEvent` sentinel is emitted instead of individual events. The batch handler in `extension.ts` treats this the same as a HIGH-priority change — it calls `runRescan()`.

**Sort order:** When the batch is dispatched to handlers, events are sorted HIGH → MEDIUM → LOW. This is informational (the handler checks priority regardless), but it keeps log output predictable.

---

## Lifecycle and Disposal

```typescript
// extension.ts — activation
const fileWatcher = new FileWatcherManager();
container.register(fileWatcher);          // disposed on deactivation

const fsWatcher = vscode.workspace.createFileSystemWatcher(pattern);
container.register(fsWatcher);            // disposed on deactivation

const batchSub = fileWatcher.onBatch(handler);
container.register(batchSub);            // unsubscribes handler on deactivation

fileWatcher.start();                      // begins accepting events
```

On `deactivate()`, `container.dispose()` calls `dispose()` on all registered objects in registration order. `FileWatcherManager.dispose()` calls `stop()` → `flush()` (processes any pending events synchronously) → clears handlers and pending map.

---

## Log Output

All log lines from the watcher use the standard Roadie Output channel (View → Output → Roadie).

| Condition | Level | Message |
|---|---|---|
| Watcher started | DEBUG | `File watcher active — watching dependency and config files` |
| Dependency/config file changed | INFO | `File watcher: package-lock.json changed — re-analysing…` |
| Batch overflow | INFO | `File watcher: batch overflow — running full rescan…` |
| Re-analysis complete | INFO | `File watcher: re-analysis complete — N commands` |
| `.github/` files updated | INFO | `File watcher: .github/ updated — .github/copilot-instructions.md` |
| `.github/` files unchanged | DEBUG | `File watcher: .github/ files unchanged` |
| Re-analysis error | ERROR | `File watcher: re-analysis failed` + error detail |

---

## What Triggers a Re-analysis (Complete Reference)

| Trigger | Automatic? | How |
|---|---|---|
| Lock file created/changed/deleted | ✅ Yes | File watcher → DEPENDENCY_CHANGE |
| `package.json` changed | ✅ Yes | File watcher → DEPENDENCY_CHANGE |
| `tsconfig.json` / build config changed | ✅ Yes | File watcher → CONFIG_CHANGE |
| > 1000 file changes at once (e.g. git checkout) | ✅ Yes | FullRescanEvent sentinel |
| Extension startup | ✅ Yes | Startup analysis in `activate()` |
| `Roadie: Initialize` command | Manual | Runs full analysis + file generation |
| `Roadie: Rescan Project` command | Manual | Runs full analysis only (no file generation) |

---

## Package Manager Detection (How Auto-Fix Works)

`dependency-scanner.ts` detects the package manager by checking for lock files in this order:

```typescript
if (await exists(path.join(root, 'pnpm-lock.yaml'))) return 'pnpm';
if (await exists(path.join(root, 'yarn.lock')))       return 'yarn';
if (await exists(path.join(root, 'bun.lockb')))       return 'bun';
return 'npm'; // fallback
```

The detected package manager is baked into every command string stored in `project_commands` (e.g. `pnpm run build`, `npm run test`).

**Auto-correction scenario:**

1. Project was initialized with npm (`package-lock.json` present) → Roadie stored `npm run …` commands
2. Developer runs `pnpm import` — `pnpm-lock.yaml` appears in the workspace root
3. File watcher detects `pnpm-lock.yaml` creation → `DEPENDENCY_CHANGE` (HIGH)
4. Batch fires after 500 ms debounce → `runRescan()` called
5. `ProjectAnalyzer.analyze()` re-runs → `detectPackageManager()` returns `'pnpm'`
6. `saveCommands()` deletes all rows and re-inserts with `pnpm run …`
7. `FileGenerator.generateAll()` detects content change → rewrites `.github/copilot-instructions.md`

The correction is fully automatic. No manual rescan needed.

---

## FileWatcherManager Public API

```typescript
class FileWatcherManager {
  constructor(config?: Partial<FileWatcherConfig>)

  // Feed a raw VS Code FileSystemWatcher event into the manager.
  handleFileEvent(filePath: string, eventType: 'create' | 'change' | 'delete'): void

  // Subscribe to debounced, classified batches. Returns a Disposable.
  onBatch(handler: BatchHandler): Disposable

  // Force-flush pending events (bypasses debounce timer).
  flush(): void

  start(): void   // Begin accepting events
  stop(): void    // Stop + flush
  dispose(): void // stop + clear all handlers and pending

  isWatching(): boolean
  getStatus(): { watching: boolean; pendingCount: number; totalEvents: number }
}

interface FileWatcherConfig {
  debounceMs: number;    // default: 500
  maxBatchSize: number;  // default: 1000
}

type BatchPayload = FileChangeEvent[] | [FullRescanEvent];

interface FileChangeEvent {
  filePath: string;
  eventType: 'create' | 'change' | 'delete';
  classifiedAs: ChangeType;   // 'DEPENDENCY_CHANGE' | 'CONFIG_CHANGE' | 'USER_EDIT' | 'SOURCE_ADDITION' | 'OTHER'
  priority: 'HIGH' | 'MEDIUM' | 'LOW';
  triggers: string[];
  timestamp: Date;
}

interface FullRescanEvent {
  type: 'FULL_RESCAN';
  eventCount: number;
  timestamp: Date;
}
```
