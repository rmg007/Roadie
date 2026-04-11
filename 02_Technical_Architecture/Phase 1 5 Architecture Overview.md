# 🏗️ Phase 1.5 Architecture Overview

## How Passive Mode Works: Data Flow, State Management, Integration

**⚠️ CRITICAL REVIEW UPDATES:** See the [📋 Comprehensive Review page](https://www.notion.so/Comprehensive-Review-Phase-1-Phase-1-5-Specs-33fc821ae63c815bb5e4c8d75467ba7d?pvs=21) for corrections needed:

- **C1:** Chokidar → VS Code FileSystemWatcher (CRITICAL — entire API restructuring)
- **C2:** Configuration defaults false (not true)
- **C3:** Learning DB size 10MB (not 100MB)
- **A3:** Single SQLite database (not two)

---

## Executive Summary

**Phase 1.5 makes Roadie truly "invisible"** by automating project configuration generation without requiring user action. Unlike Phase 1 (user initiates workflows via chat), Phase 1.5 runs silently in the background, watching the codebase and maintaining up-to-date AI-configuration files.

**Core idea:** When a developer changes dependencies or structure, Roadie detects it, updates its internal project model, and regenerates .github/ files—all without a single chat message. The developer never thinks about Roadie; it just works.

**Key design principle:** Respect human edits. Developers can modify generated files. Roadie detects these changes (via hashes) and merges them intelligently.

---

## System Architecture

### Layer 1: File System Monitoring

```jsx
┌──────────────────────────────────────────────────────────────┐
│ File System (VS Code workspace)                             │
├──────────────────────────────────────────────────────────────┤
│ ├─ package.json (dependencies)                              │
│ ├─ src/ (source code)                                       │
│ ├─ tsconfig.json (TypeScript config)                        │
│ ├─ .github/ (generated files)                               │
│ │  ├─ copilot-instructions.md (Roadie-generated)           │
│ │  ├─ copilot/ (path-specific instructions)                │
│ │  ├─ agents/ (agent definitions)                          │
│ │  ├─ workflows/ (CI/CD)                                   │
│ │  └─ .roadie/ (Roadie's private data)                    │
│ │     └─ project-model.db (SQLite, unified DB)             │
│ └─ ... (other source files)
└──────────────────────────────────────────────────────────────┘
         ↓
    File Watcher
    (VS Code FileSystemWatcher - NOT chokidar)
    ├─ Watches glob patterns
    ├─ Debounces events (500ms)
    ├─ Classifies changes
    └─ Dispatches to updaters
```

### Layer 2: Project Model (Persistent State)

```
┌──────────────────────────────────────────────────────────────┐
│ Project Model (In-Memory + SQLite)                          │
├──────────────────────────────────────────────────────────────┤
│ At Extension Activation:                                     │
│  1. Load from SQLite (or build from scratch if missing)     │
│  2. Validate against current file system                    │
│  3. Reconcile if out of sync                                │
│  4. Expose query APIs                                       │
│                                                              │
│ During Runtime:                                              │
│  - File Watcher sends change notifications                  │
│  - Project Model Updater applies incremental updates        │
│  - Triggers regeneration of affected generators             │
│  - Caches results in SQLite                                 │
│                                                              │
│ At Extension Shutdown:                                       │
│  - Flush pending writes                                     │
│  - Close SQLite connections                                 │
└──────────────────────────────────────────────────────────────┘
         ↓
    Query APIs
    ├─ getTechStack()
    ├─ getSourceStructure()
    ├─ getDependencies()
    ├─ detectFramework()
    └─ ... (used by generators)
```

### Layer 3: File Generators

```
┌──────────────────────────────────────────────────────────────┐
│ File Generators (8 types, triggered by model changes)       │
├──────────────────────────────────────────────────────────────┤
│ For each generator:                                          │
│  1. Project Model query → get context                       │
│  2. Apply template → generate content                       │
│  3. Check if changed (diff-before-write)                    │
│  4. Write to file (with section markers)                    │
│  5. Log to learning DB                                      │
│                                                              │
│ Generators:                                                  │
│  - Copilot Instructions (1 file: .github/copilot-*.md)     │
│  - Path Instructions (N files: .github/copilot/{path}.md)   │
│  - Agents (N files: .github/agents/*.yaml)                  │
│  - Skills (N files: .github/skills/*.md)                    │
│  - Hooks (N files: .github/hooks/*.sh)                      │
│  - Workflows (N files: .github/workflows/*.yml)             │
│  - Templates (N files: .github/ISSUE_TEMPLATE/*.md)         │
│  - AGENTS.md (1 file: .github/AGENTS.md)                    │
└──────────────────────────────────────────────────────────────┘
         ↓
    Section Manager
    ├─ Detects ownership markers
    ├─ Computes hashes
    ├─ Merges human edits
    └─ Writes intelligently
```

### Layer 4: Edit Tracking & Learning

```jsx
┌──────────────────────────────────────────────────────────────┐
│ Edit Tracking & Learning System                             │
├──────────────────────────────────────────────────────────────┤
│ Edit Tracker:                                                │
│  - Monitors .github/ files for modifications                │
│  - Computes diffs (before/after)                            │
│  - Stores snapshots                                         │
│                                                              │
│ Learning Database (shared SQLite with project model):       │
│  - Edit history (what humans changed)                       │
│  - Workflow outcomes (did workflows succeed?)               │
│  - Discovered patterns (what conventions does this project?)│
│  - Anti-patterns (what bugs did we find?)                   │
│                                                              │
│ Used By:                                                     │
│  - File Generators (improve templates based on history)     │
│  - Workflow Engine (understand project conventions)         │
│  - Agent Spawner (inject learned context)                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Data Flow Examples

### Example 1: Developer Adds a Dependency

```
Step 1: Developer updates package.json
  package.json: "react": "18.2" → "19.0"
     ↓
Step 2: File Watcher detects change
  Event: file-changed("package.json")
  Classification: DEPENDENCY_CHANGE
     ↓
Step 3: Project Model Updater applies change
  Update: projectModel.dependencies["react"] = "19.0"
  Trigger: generators that depend on "react"
     ↓
Step 4: Affected Generators regenerate
  Copilot Instructions: "Update React patterns to v19"
  Path Instructions: "Components guide uses React 19"
  Agent Definitions: "Feature agent aware of React 19 hooks"
     ↓
Step 5: Section Manager writes files
  - Read existing file
  - Find Roadie-owned sections (by markers)
  - Preserve human edits outside those sections
  - Write new content if changed
     ↓
Step 6: Edit Tracker logs the generation
  - File: .github/copilot-instructions.md
  - Change type: DEPENDENCY_UPDATE
  - Diff: [before, after]
  - Timestamp: now
     ↓
Step 7: Learning Database stores pattern
  - Pattern: "React 19 requires different hook usage"
  - Frequency: first time seeing React 19
  - Used in future for similar updates
```

### Example 2: Developer Edits Generated File

```
Step 1: Developer modifies .github/copilot-instructions.md
  Edits the file in VS Code
  Changes content inside and outside Roadie sections
     ↓
Step 2: Edit Tracker detects modification
  Event: file-changed(".github/copilot-instructions.md")
  Classification: USER_MODIFICATION (not Roadie-generated)
     ↓
Step 3: Edit Tracker computes diff
  Before (from last snapshot): [original content]
  After (current file): [human-edited content]
  Diff: [changes made by developer]
     ↓
Step 4: Learning Database stores edit
  File: .github/copilot-instructions.md
  Type: USER_EDIT
  Sections changed: [list of sections]
  Diff: [before/after]
     ↓
Step 5: Later, something triggers regeneration (e.g., tech stack change)
  New content generated
  But developer's edits are preserved (because of section markers)
     ↓
Step 6: Section Manager merges
  1. Parse ownership markers
  2. Identify Roadie-owned vs user-owned sections
  3. Preserve user sections exactly
  4. Update Roadie sections
  5. Write merged result
```

### Example 3: Project Structure Changes

```
Step 1: Developer adds new src/components/ directory
  mkdir src/components/
  touch src/components/Button.tsx
     ↓
Step 2: File Watcher detects structure change
  Event: directory-created("src/components/")
  Event: file-created("src/components/Button.tsx")
  Classification: STRUCTURE_CHANGE
     ↓
Step 3: Project Model Updater updates source structure
  Update: projectModel.sourceStructure["src/components"] = {...}
  Trigger: Path Instructions generator (generates per-directory guides)
     ↓
Step 4: Path Instructions Generator creates new file
  Create: .github/copilot/src/components.md
  Content: "# Components Directory\nFocus on reusable UI components..."
     ↓
Step 5: Section Manager writes new file
  File doesn't exist yet, so just write it
  Add Roadie section markers (marks this as Roadie-generated)
     ↓
Step 6: Learning Database stores creation
  New file created
  Pattern: "When new directory created, generate guide"
```

---

## State Machine: Startup, Runtime, Shutdown

### Activation (VS Code opens Roadie extension)

```
1. INITIALIZE
   ├─ Register commands (roadie.init, roadie.rescan, etc.)
   ├─ Load configuration (from VS Code settings)
   └─ Create status bar

2. LOAD PROJECT MODEL
   ├─ Check if .github/.roadie/project-model.db exists
   ├─ YES → Load from SQLite
   ├─ NO → Build from scratch (scan file system)
   ├─ Validate against current files (reconcile if needed)
   └─ Log to learning DB: "Project model loaded/built"

3. START FILE WATCHER
   ├─ Register glob patterns
   ├─ Handle watcher errors gracefully (fallback to polling)
   └─ Ready for file changes

4. REGISTER CHAT PARTICIPANT
   ├─ Chat Participant handler ready (Phase 1)
   └─ User can now send messages

5. RESTORE PREVIOUS STATE (if any)
   ├─ Load workflow history from learning DB
   ├─ Restore any interrupted workflows
   └─ Report status in output channel

State: READY
```

### Runtime (Extension is running)

```
Wait for Events:

┌─ FILE CHANGE EVENT
│  ├─ File Watcher detects change
│  ├─ Debounce (wait for similar events to batch)
│  ├─ Classify change type (dependency, structure, source, etc.)
│  ├─ Project Model Updater applies change
│  ├─ Triggers generators (if needed)
│  ├─ Writes files
│  └─ Logs to learning DB
│
├─ USER CHAT MESSAGE
│  ├─ Intent Classifier (Phase 1)
│  ├─ Workflow Engine (Phase 1)
│  ├─ Agent Spawner (Phase 1)
│  ├─ Injects project model context (Phase 1.5)
│  ├─ Logs outcome to learning DB
│  └─ Updates project model if needed (e.g., fixed bug)
│
├─ USER COMMAND
│  ├─ roadie.init → Initialize empty .github/.roadie/
│  ├─ roadie.rescan → Rebuild project model from scratch
│  ├─ roadie.reset → Clear all generated files
│  └─ Logs action to learning DB
│
└─ SCHEDULED MAINTENANCE (optional)
   ├─ Prune learning DB (remove old entries)
   └─ Validate project model consistency

State: READY (until shutdown)
```

### Deactivation (VS Code closing or extension disabled)

```
1. FLUSH PENDING WRITES
   ├─ Write any uncommitted project model changes
   ├─ Write any pending edit tracking entries
   └─ Flush learning DB

2. CLOSE FILE WATCHER
   ├─ Unregister all glob patterns
   ├─ Stop listening to file system
   └─ Handle pending events (ignore them)

3. CLOSE DATABASE CONNECTIONS
   ├─ Close project-model.db connection
   ├─ Close learning.db connection
   └─ Ensure data is persisted

4. UNREGISTER HANDLERS
   ├─ Unregister chat participant
   ├─ Unregister commands
   └─ Clean up subscriptions

State: INACTIVE
(Next activation will load from SQLite)
```

---

## Integration with Phase 1

### Project Model Changes

**Phase 1 (In-memory only):**

```tsx
class ProjectModel {
  tech_stack: TechStackEntry[];
  directory_structure: DirectoryNode;
  // ... built lazily from file system
}
```

**Phase 1.5 (Persistent + Incremental):**

```tsx
class ProjectModel {
  tech_stack: TechStackEntry[];
  directory_structure: DirectoryNode;
  // ... loaded from SQLite at activation
  // ... incrementally updated by file watcher
  
  // NEW: Query APIs for generators
  getTechStack(): TechStackEntry[]
  getSourceStructure(path?: string): DirectoryNode
  getDependencies(): Map<string, string>
  getFramework(): Framework
  getTestFramework(): Framework
  // ... more queries
  
  // NEW: Persisted to SQLite
  async saveToDb(): Promise<void>
  async loadFromDb(): Promise<void>
  async reconcileWithFileSystem(): Promise<void>
}
```

### Workflow Engine Changes

**Phase 1:** Workflows execute, results shown to user

**Phase 1.5 (additions):**

- Log workflow outcomes to learning DB
- Extract discovered patterns (e.g., "this project uses Jest for tests")
- Pass patterns back to project model
- Enrich future prompts with discovered context

### Agent Spawner Changes

**Phase 1:** Injects minimal context

**Phase 1.5 (richer context):**

- Pull from persistent project model (not just lazy detection)
- Include discovered patterns from learning DB
- Inject edit history (what developers changed in the past)
- Reference existing .github/ conventions

---

## Error Handling & Degradation

### SQLite Database Corruption

```
Detect:
  Open .github/.roadie/project-model.db
  If error → CORRUPTED

Recovery:
  1. Rename corrupted DB (backup)
  2. Rebuild project model from scratch
  3. Rescan file system
  4. Regenerate all files
  5. Log incident to learning DB (if possible)
  6. Show warning to user

Result:
  Extension continues working (Phase 1 mode until model rebuilt)
```

### File Watcher Crash

```
Detect:
  File watcher emits error event
  Cannot re-watch directory

Fallback:
  1. Use polling instead of native watchers
  2. Check file system every N seconds
  3. Reduce frequency for large workspaces
  4. Log warning (watching via polling, performance impact)

Recovery:
  Can manually trigger rescan via command
```

### Permission Errors

```
Detect:
  Cannot read file (permission denied)
  Cannot write file (permission denied)

Action:
  1. Log error
  2. Skip file (don't fail whole extension)
  3. Show warning in output channel
  4. Retry on next file change (permission might be fixed)

Result:
  Extension continues, but file is not analyzed/generated
```

### Out-of-Sync Detection

```
At startup:
  1. Load project model from SQLite
  2. Scan current file system
  3. Compare (tech stack, dependencies, source structure)
  
If mismatch detected:
  1. Log warning
  2. Automatically reconcile (update model)
  3. Trigger regeneration of affected files
  4. No user action needed
```

---

## Performance Considerations

### File Watcher

- **API:** VS Code FileSystemWatcher (workspace-trust aware, Remote Development compatible)
- **Fallback:** Polling if FileSystemWatcher fails
- **Limit:** Max 5000 watched paths (configurable)
- **Debounce:** 500ms (batch rapid changes)

### Project Model

- **Load time:** <500ms (load from SQLite)
- **Validation:** <1s (scan current files)
- **Query time:** <100ms (in-memory, indexed queries)

### File Generators

- **Template compilation:** <50ms per file
- **Diff computation:** <100ms per file
- **Write:** <100ms per file
- **Total:** All 8+ generators < 2s per trigger

### Learning Database

- **Size:** Keep < 10MB (prune old entries)
- **Query time:** <200ms (indexed queries)
- **Write:** Async (non-blocking)

---

## Configuration Options (Phase 1.5)

```tsx
// New settings (extend Phase 1 settings)
roadie.editTracking: boolean         // default: false
roadie.workflowHistory: boolean      // default: false
roadie.autoCommit: boolean           // default: false
roadie.fileWatcherTimeout: number    // default: 5000ms
roadie.maxWatchedPaths: number       // default: 5000
roadie.learningDbRetention: number   // default: 90 days
```

---

## Backward Compatibility: Phase 1.5 Activation Boundary

**When does Phase 1.5 activate?**

1. Automatically when `.github/.roadie/project-model.db` exists
2. On first Phase 1 workflow execution (creates the database)
3. Falls back to Phase 1 if developer deletes `.github/.roadie/`

**Fresh Install Behavior:**

- Extension starts in Phase 1-only mode
- All Phase 1.5 settings default to `false` (opt-in)
- When first workflow runs, Phase 1.5 activates transparently
- Design principle: "Every boolean defaults to false. Roadie does nothing the developer hasn't explicitly consented to."

## Next Steps

This architecture shows how Phase 1.5 extends Phase 1 without breaking anything. The key insight:

**Phase 1 = Active (user initiates) + Phase 1.5 = Passive (automatic) = Complete invisibility**

**See:** 📋 [Comprehensive Review Page](https://www.notion.so/Comprehensive-Review-Phase-1-Phase-1-5-Specs-33fc821ae63c815bb5e4c8d75467ba7d?pvs=21) for complete list of corrections and implementation priorities.

Next: Detailed specs with corrections applied (Project Model Persistence, File Generator Manager, Section Manager with append-below merge, etc.)