# 🔄 Phase 1.5 Implementation Specification — Master Index

## Passive Mode: File Watching, Persistent Model, Silent Generation

**Status:** ✅ IMPLEMENTED — Phase 1.5 fully built and integrated as of 2026-04-12  

**Scope:** Passive Mode (File Watcher, Persistent Model, File Generation, Edit Tracking, Learning DB)  

**Build on Top Of:** Phase 1 (Active Mode, Chat Participant, Workflow Engine)  

**Target Audience:** AI Coding Agents, Senior Engineers  

**Estimated Build Time:** 41-54 hours  

**Critical Complexity:** Section ownership + merge logic (most error-prone)  

---

## What Phase 1.5 Adds

### Core Capabilities

**File System Watcher**

- Monitors workspace for changes (dependencies, source structure, config)
- Debounces events, classifies change types, dispatches to updaters
- Graceful degradation if watcher crashes

**Persistent Project Model**

- Survives across VS Code sessions (SQLite storage)
- Incrementally updated instead of rebuilt (performance)
- Loaded at activation, reconciled with file system

**Silent File Generation**

- Produces .github/ files from project model without user action
- Respects section ownership (detects human edits)
- Diff-before-write (only write if changed)
- 9 file types: instructions, agents, skills, hooks, workflows, templates, AGENTS.md, dot-roadie config, codebase dictionary

**Edit Tracking**

- Detects when developers modify generated files
- Computes before/after diffs
- Stores snapshots for learning system

**Learning Database**

- SQLite storage for edit tracking, workflow outcomes, discovered patterns
- Used to improve generated content over time
- Pruning and size management

### Integration with Phase 1

- Phase 1 workflows enhanced with persistent model context
- Workflow outcomes logged to learning DB
- Discovered patterns (from bug fixes, code reviews) persisted
- No breaking changes to Phase 1 APIs (backward compatible)

### What Phase 1.5 Does NOT Include

- ❌ MCP server (Phase 2)
- ❌ Cross-tool compatibility (Phase 2)
- ❌ Section sidebar UI (Phase 2)
- ❌ Multi-ecosystem support (Phase 2+)

---

## 📚 Specification Pages

### Foundation & Architecture

1. **🏗️ Phase 1.5 Architecture Overview** — Big picture, data flow, state diagram, integration points

### New Module Specifications

1. **👁️ File Watcher Manager** — Glob patterns, debouncing, classification, error handling
2. **💾 Project Model Persistence** — SQLite schema, startup reconciliation, incremental updates
3. **📄 File Generator Manager** — Orchestration, trigger mapping, deferred writes, parallel execution
4. **🏷️ Section Manager** (CRITICAL) — Markers, hashing, append-below merge, edge cases
5. **✏️ Edit Tracker** — Edit detection, diff computation, snapshot storage
6. **📚 Learning Database** — SQLite schema, retention, pruning, section hashes
7. **📖 Codebase Dictionary** — Entity annotation, SQLite storage, relationship extraction, context surfacing

### File-Specific Generators

1. **📦 File-Specific Generator Templates (All 9)** — Copilot Instructions, Path Instructions, Agent Definitions, Skills, Hooks, Workflows, Templates, AGENTS.md, Codebase Dictionary

### Phase 1 Integration & Configuration

1. **🔄 Phase 1 Integration: Module Changes** — Exactly what changes in each Phase 1 module
2. **⚙️ Phase 1.5 Configuration Schema** — All settings with types, defaults, validation
3. **📊 Phase 1.5 Module Build Order & Dependencies** — 10-step build sequence with verification gates

---

## 🎯 How to Use This Specification

### For AI Coding Agents

**Setup (Day 1)**

1. You've completed Phase 1 ✓
2. Read this Master Index (you're reading it)
3. Read "Phase 1.5 Architecture Overview" (big picture)
4. Read "Implementation Patterns" from Phase 1 (still applies)

**Build (Days 2-15)**

1. Go to "Module Build Order & Dependencies"
2. Follow M16 → M23 → M15 → M22 → M24 → M19 → M21 (build sequence)
3. Each module has a "Build Prompt" ready to paste
4. After each module, run verification criteria
5. Section Manager is most complex — go slow, ask questions

**If You Get Stuck**

1. Check Phase 1 "Implementation Patterns & Standards" (still applies)
2. Look at relevant spec page for detailed algorithms & pseudocode
3. Check "Architecture Overview" if confused about data flow
4. Ask for clarification before implementing

### For Code Reviewers

1. **Module Review:** Compare against spec + Phase 1 patterns
2. **Integration Review:** Verify Phase 1 backward compatibility
3. **Section Manager Review:** Extra scrutiny (merge logic is complex)
4. **Testing:** Run file watcher tests + edit tracking tests + generation tests

### For Product Owners

- **Timeline:** Phase 1 complete + 15 more modules = 41-54 hours
- **Critical Path:** Project Model Persistence (M16) was blocker for all generators — now complete
- **Risk:** Section Manager (M22) was most complex — now complete and tested
- **Demo:** Generators working — auto-generated .github/ files are functional

---

## 📊 Architecture at a Glance

```
Phase 1 (Active)
├─ Chat Participant (user prompts)
├─ Intent Classifier
├─ Workflow Engine
└─ Agent Spawner (executes workflows)

Phase 1.5 (Passive) — NEW
├─ File Watcher
│  ├─ Monitors .github/, package.json, tsconfig.json, src/, etc.
│  ├─ Debounces, classifies changes
│  └─ Dispatches to Project Model Updater
├─ Project Model Persistence
│  ├─ SQLite DB (.github/.roadie/project-model.db)
│  ├─ Loaded at activation
│  ├─ Incremental updates from watcher
│  └─ Exposes query APIs
├─ File Generators (triggered by model changes)
│  ├─ Copilot Instructions
│  ├─ Path Instructions (src/, components/, etc.)
│  ├─ Agent Definitions
│  ├─ Skills
│  ├─ Hooks
│  ├─ Workflows
│  ├─ Templates
│  ├─ AGENTS.md
│  └─ Codebase Dictionary
├─ Section Manager (preserves human edits)
│  ├─ Detects section ownership markers
│  ├─ Computes hashes of Roadie-owned sections
│  └─ Merges human edits with regenerated content
├─ Edit Tracker
│  ├─ Detects modifications to .github/ files
│  ├─ Computes diffs
│  └─ Stores snapshots
└─ Learning Database
   ├─ Edit history
   ├─ Workflow outcomes
   ├─ Discovered patterns
   └─ Generates insights for content improvement

Phase 1 Integration Points
├─ Project Model: now loads from SQLite, receives updates from watcher
├─ Workflow Engine: logs outcomes to learning DB, passes patterns to model
└─ Agent Spawner: injects richer context from persistent model
```

---

## 🚨 Critical Complexity Areas

### Section Manager (Module M22)

This is the most error-prone part of Phase 1.5. Specified with extreme precision:

- Marker format (exact syntax for JSON, YAML, Markdown, TypeScript)
- Hash computation (what gets hashed?)
- Merge algorithm: **"append below"** — when human has edited inside a Roadie section, append new Roadie content below human edits with `<!-- roadie:merged:timestamp -->` separator. Never overwrite, never silently discard.
- Conflict resolution: always append-below for edited sections; skip file entirely if markers removed
- Test cases (every scenario covered, including concurrent edit race conditions)
- **Data loss prevention:** Add deferred write logic — if file is open and unsaved in VS Code editor, queue the write until the file is saved (see TAD §7.4)

**Risk Level:** HIGH — test coverage must be comprehensive

### File Watcher (Module M15)

Edge cases that will cause bugs:

- Large workspaces (10k+ files) — polling fallback
- Git checkouts (many files change at once) — smart batch processing: <100 events process individually, 100-1000 batch then consolidate, >1000 full rescan
- Watcher crash — graceful restart
- Permission errors — skip file, log, continue
- Symlinks — handle correctly
- **Startup reconciliation** — on activation, scan watched files and compare modification timestamps against project model's lastAnalyzed timestamps. Process changes that occurred while VS Code was closed.
- **No directory events** — VS Code FileSystemWatcher does NOT fire addDir/unlinkDir. Infer directory changes from file creation/deletion paths.

**Risk Level:** MEDIUM — edge cases must be tested

### Project Model Persistence (Module M16)

Startup/shutdown correctness is critical:

- Load from SQLite
- Validate against current file system
- Reconcile if out of sync
- Flush on shutdown
- Handle corruption

**Risk Level:** MEDIUM — must be robust

---

## 📋 Module Count & Build Time

| Phase | Modules | Files | Estimated Hours | Risk Level |
| --- | --- | --- | --- | --- |
| Phase 1 | 14 | 28 | 33.5 | Medium |
| **Phase 1.5** | **~15** | **~30** | **41-54** | **High** |
| **Total** | **~29** | **~58** | **~75-88** | — |

**Phase 1.5 Risk Drivers:**

- Section Manager complexity (M22, ~7 modules)
- 9 different file generators (generator sub-modules including Codebase Dictionary)
- SQLite schema design (M16)
- File watcher edge cases (M15)

---

## 🔗 Quick Links

| Need | Page |
| --- | --- |
| I want to understand Phase 1.5 big picture | Architecture Overview |
| I'm building the file watcher | File Watcher Manager |
| I'm persisting the project model | Project Model Persistence |
| How do section ownership markers work? | Section Manager |
| I'm building a file generator | File Generator Manager + specific generator page |
| What changed in Phase 1 modules? | Phase 1 Integration: Module Changes |
| What's the build sequence? | Module Build Order & Dependencies |
| I need the SQLite schema for learning DB | Learning Database page |

---

## 🗺️ Module-to-Roadmap Milestone Mapping

The Phase 1.5 specs use internal module IDs (M15-M24) that don't map 1:1 to Roadmap milestones (M15-M20). Here's the mapping:

| Spec Module ID | Spec Module Name | Roadmap Milestone | Roadmap Description |
| --- | --- | --- | --- |
| M16 | Project Model Persistence | M15 | File Watcher + Incremental Model Updates |
| M15 | File Watcher Manager | M15 | File Watcher + Incremental Model Updates |
| M23 | Learning Database | M18 | Learning Database + Edit Tracking |
| M22 | Section Manager | M16 | Automatic File Regeneration |
| M19 | File Generator Manager | M16 | Automatic File Regeneration |
| M21 | Edit Tracker | M18 | Learning Database + Edit Tracking |
| Generators | 9 Generator Sub-modules | M17 + M19 | Pattern Detection + Extended File Gen |
| M24 | Codebase Dictionary | M17 | Pattern Detection |
| Integration | Phase 1 Integration | M20 | Sidebar View + Status Dashboard |

> **Note:** Roadmap milestones group multiple spec modules together. When building, follow the **Phase 1.5 Module Build Order** page (which uses the spec module IDs), not the Roadmap milestone sequence. The Roadmap is for product planning; the Build Order is for implementation.
> 

---

## ✅ Specification Status

**Phase 1.5 Master Index:** ✅ COMPLETE

**Detailed Specification Pages: ALL COMPLETE**

- [x]  Architecture Overview ✅
- [x]  File Watcher Manager ✅
- [x]  Project Model Persistence ✅
- [x]  File Generator Manager ✅
- [x]  Section Manager (CRITICAL) ✅
- [x]  Edit Tracker ✅
- [x]  Learning Database ✅
- [x]  File-Specific Generator Templates (All 9) ✅
- [x]  Codebase Dictionary ✅
- [x]  Phase 1 Integration Guide ✅
- [x]  Configuration Schema ✅
- [x]  Module Build Order & Dependencies ✅

**All 12 specification pages are complete. Phase 1.5 is fully implemented.**

---

## ⚠️ Backward Compatibility Requirements

**Phase 1 must work perfectly without Phase 1.5.** Phase 1.5 is additive and config-driven.

### Activation Boundary

- Phase 1.5 features activate when `.github/.roadie/project-model.db` exists (created by first workflow run) AND Phase 1.5 code is present in the extension.
- If a developer deletes `.github/.roadie/`, the extension falls back to Phase 1 behavior (lazy project model, no file watching, no generation).
- All Phase 1.5 settings default to `false` (opt-in).
- The File Watcher, Edit Tracker, and Learning Database check `isPhase15Active()` before initializing.

### Interface Extension (Not Modification)

```tsx
// Phase 1 interface (UNCHANGED — do not modify)
interface ProjectModel {
  getTechStack(): TechStackEntry[];
  toContext(options?: ContextOptions): ProjectContext;
  // ... all Phase 1 methods
}

// Phase 1.5 EXTENDS (does not replace)
interface PersistentProjectModel extends ProjectModel {
  saveToDb(): Promise<void>;
  loadFromDb(): Promise<void>;
  reconcileWithFileSystem(): Promise<ReconciliationResult>;
}
```

### Single Database File

All data (project model + learning data) lives in ONE SQLite database: `.github/.roadie/project-model.db`. Do NOT create a separate `learning.db`.

### Performance Requirements

- All generators: < 2s total per trigger
- File watcher event latency: < 1s (event → model update → generator trigger)
- No blocking operations in passive mode — all file I/O uses async APIs (`fs.promises`)
- No synchronous `fs.readFileSync` or `fs.writeFileSync` in watcher or generator code paths

---

## 🎬 Next Steps

1. ✅ Phase 1 spec is COMPLETE
2. ✅ Phase 1.5 spec is COMPLETE (all 11 pages written)
3. ✅ Phase 2 spec is COMPLETE (all 7 pages written)
4. ✅ Phase 1 IMPLEMENTED
5. ✅ Phase 1.5 IMPLEMENTED
6. 🤖 **NEXT:** Build Phase 2 (follow Phase 2 Build Order)
7. 🧪 Integration testing at each phase boundary
8. 🚀 Ship v1.0

---

**Created:** April 2026  

**Status:** ✅ Phase 1 & Phase 1.5 IMPLEMENTED — Phase 2 next  

**Last Updated:** 2026-04-12

## Related Specification Pages

- [Phase 1.5 Architecture Overview](Phase%201%205%20Architecture%20Overview.md)
- [File Watcher Manager Specification](../04_Implementation_Specs_Phase_1.5/File%20Watcher%20Manager%20Specification.md)
- [Section Manager Specification](../04_Implementation_Specs_Phase_1.5/Section%20Manager%20Specification.md)
- [Issues & Fixes Status Tracking Dashboard](Issues%20&%20Fixes%20Status%20Tracking%20Dashboard.md)
- [Implementation Roadmap & Weekly Checklist](../01_Product_Strategy/Implementation%20Roadmap%20&%20Weekly%20Checklist.md)
- [Phase 1.5 Master Review & Execution Guide](Phase%201%205%20Master%20Review%20&%20Execution%20Guide.md)
- [Project Model Persistence Specification](../04_Implementation_Specs_Phase_1.5/Project%20Model%20Persistence%20Specification.md)
- [File Generator Manager Specification](../04_Implementation_Specs_Phase_1.5/File%20Generator%20Manager%20Specification.md)
- [Edit Tracker Specification](../04_Implementation_Specs_Phase_1.5/Edit%20Tracker%20Specification.md)
- [Learning Database Specification](../04_Implementation_Specs_Phase_1.5/Learning%20Database%20Specification.md)
- [Codebase Dictionary Specification](../04_Implementation_Specs_Phase_1.5/Codebase%20Dictionary%20Specification.md)
- [Phase 1.5 Module Build Order & Dependencies](../03_Implementation_Specs_Phase_1/Phase%201%205%20Module%20Build%20Order%20&%20Dependencies.md)
- [File-Specific Generator Templates (All 9)](../04_Implementation_Specs_Phase_1.5/File-Specific%20Generator%20Templates.md)