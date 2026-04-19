# 📄 File Generator Manager Specification

## Orchestrates Generation of All .github/ Files, Triggers on Model Changes, Manages Write Pipeline

---

## Module Identity

**Module ID:** M19

**File Location:** `src/generator/file-generator-manager.ts`

**Depends On:** PersistentProjectModel (M16), Section Manager (M22), Database (M5)

**Used By:** Extension activation, File Watcher (via model change events)

**Complexity:** Medium (orchestration logic, 9 generator sub-modules)

**Estimated Build Time:** 4-5 hours (manager only; generators are separate modules)

**Implementation Status:** ✅ COMPLETE — Implemented as of 2026-04-12

---

## Responsibility

The File Generator Manager is the orchestrator that:

1. **Subscribes** to project model change events
2. **Determines** which files need regeneration based on what changed
3. **Calls** the appropriate generator sub-module for each file
4. **Passes** generated content through the Section Manager for merge/write
5. **Logs** generation events to the Learning Database
6. **Enforces** performance budgets (all generators < 2s total)

It does NOT generate content itself — that's the job of the 9 generator sub-modules (including Codebase Dictionary).

---

## Generation Workflow (v1.0.0)

```
init → scan → template selection → file write
```

**Step 1 — init:** FileGeneratorManager is initialized during extension activation. It subscribes to `PersistentProjectModel` change events and registers all generator sub-modules.

**Step 2 — scan:** On model change or `generateAll()` call, the manager reads the `ProjectModelDelta` to determine which file types are affected. IDE detection (`detectIDEs()`) is called once per session and cached.

**Step 3 — template selection:** Each triggered generator receives the full `ProjectModel` and returns `GeneratedContent` (sections array). Template selection is static in v1.0.0 — all generators produce their full template; Section Manager handles merge.

**Step 4 — file write (async):** Generated content is passed to Section Manager which:
1. Reads existing file content (if any).
2. Merges: preserves user-edited sections (`<!-- roadie:user-edit -->` markers), overwrites Roadie-owned sections.
3. Writes atomically (temp file + rename).
4. Returns `WriteResult` with `written`, `merged`, `deferred` flags.
5. If file is open in editor (`deferred: true`), write is queued for next save.

**Conflict detection:** If a user has modified a Roadie-owned section (detected by hash mismatch on owned markers), the manager logs a warning but does NOT overwrite. User edits take precedence.

---

## Templates Reference (v1.0.0)

| Template / Generator | Output File | Consumer |
|---|---|---|
| Copilot Instructions Generator | `.github/copilot-instructions.md` | GitHub Copilot |
| Agent Definitions Generator | `AGENTS.md` | Any AI agent |
| CLAUDE.md Generator | `CLAUDE.md` | Claude Code / Claude tools |
| Cursor Rules Generator | `.cursor/rules/project.mdc` | Cursor IDE |
| Path Instructions Generator | `.github/instructions/*.md` | Path-scoped context |
| Hooks Generator | `src/generator/templates/claude-hooks.ts` | Claude Code (Phase 2, not yet called) |
| Agent Definitions Template | `src/generator/templates/agent-definitions.ts` | MCP clients (Phase 2, not yet called) |
| Codebase Dictionary Generator | `.roadie/dictionary.db` (SQLite) | Internal dictionary queries |
| Scan Summary | `.roadie/last-scan.json` | Machine-readable metadata |

---

## Error Handling & Rollback (v1.0.0)

| Error Condition | Behavior |
|---|---|
| File already exists (no user edits) | Overwrite with merged content |
| File exists with user edits | Preserve user sections; overwrite Roadie sections only |
| File open in editor (unsaved) | Defer write; queue for next save via `deferredWrites` |
| Disk write fails (permissions, disk full) | Log error to Output channel; skip this file; continue with remaining |
| Template not found for file type | Log error; skip file; continue with remaining |
| Generator throws exception | Catch, log, increment error counter; continue with remaining generators |
| All generators fail | Log summary error; do not surface to user (silent degradation) |

No partial rollback is implemented — each file is written independently. A failed write to one file does not affect others.

---

## Architecture

```
Model Change Event
    │
    ▼
┌───────────────────────────┐
│ File Generator Manager     │
│                             │
│  1. What changed?           │
│  2. Which files affected?   │
│  3. Call generators          │
│  4. Pass to Section Manager │
│  5. Log to Learning DB      │
└───────┬───────────────────┘
        │
        ├─── Copilot Instructions Generator
        ├─── Path Instructions Generator
        ├─── Agent Definition Generator
        ├─── Skill Generator
        ├─── Hooks Generator
        ├─── Workflows Generator
        ├─── Templates Generator
        ├─── AGENTS.md Generator
        └─── Codebase Dictionary Generator
```

---

## Trigger-to-Generator Mapping

| Model Change Type | Generators Triggered |
| --- | --- |
| Tech stack changed (dependency add/remove/update) | Copilot Instructions, [AGENTS.md](http://AGENTS.md), Agent Definitions |
| Config changed (tsconfig, eslint, etc.) | Copilot Instructions, Path Instructions |
| Directory structure changed | Path Instructions, [AGENTS.md](http://AGENTS.md) |
| Patterns detected/updated | Copilot Instructions, Path Instructions |
| Workflow completed (first run of a type) | Agent Definitions, Skills |
| Linter/formatter detected | Hooks |
| CI structure detected | Workflows |
| Workflow completed (any type with implementation steps) | Codebase Dictionary |
| Any model change | PR Template, Issue Templates (regenerate always) |

---

## Public Interface

```tsx
interface FileGeneratorManager {
  /** Initialize: subscribe to model changes, register generators */
  initialize(model: PersistentProjectModel): void;
  
  /** Generate a specific file type on demand */
  generate(fileType: GeneratedFileType): Promise<GenerationResult>;
  
  /** Generate all files (full regeneration) */
  generateAll(): Promise<GenerationResult[]>;
  
  /** Handle a model change event (determines which files to regenerate) */
  onModelChanged(delta: ProjectModelDelta): Promise<void>;
  
  /** Dispose: unsubscribe from events */
  dispose(): void;
}

interface GenerationResult {
  fileType: GeneratedFileType;
  filePath: string;
  written: boolean;        // false if content unchanged (hash match)
  merged: boolean;         // true if append-below merge occurred
  deferred: boolean;       // true if file was open/unsaved in editor
  contentHash: string;
  durationMs: number;
}

// Each generator sub-module implements this
interface FileTypeGenerator {
  fileType: GeneratedFileType;
  /** Which model change categories trigger this generator */
  triggers: string[];
  /** Generate content from the project model */
  generate(model: ProjectModel): Promise<GeneratedContent>;
}

interface GeneratedContent {
  filePath: string;        // Relative to workspace root
  sections: GeneratedSection[];  // Passed to Section Manager
}
```

---

## Generation Pipeline

```tsx
async function runGenerationPipeline(
  generator: FileTypeGenerator,
  model: ProjectModel
): Promise<GenerationResult> {
  const startTime = performance.now();
  
  // Step 1: Generate content
  const content = await generator.generate(model);
  
  // Step 2: Check if file exists
  const existingContent = await readFileIfExists(content.filePath);
  
  // Step 3: Pass to Section Manager for merge
  const writeResult = await sectionManager.writeSectionFile(
    content.filePath,
    content.sections,
    existingContent
  );
  
  // Step 4: Check deferred (file open in editor)
  if (!writeResult.written && writeResult.deferred) {
    deferredWrites.queue(content.filePath, content.sections);
  }
  
  // Step 5: Log to Learning Database
  if (writeResult.written) {
    await learningDb.recordSnapshot(
      content.filePath,
      writeResult.finalContent,
      'roadie'
    );
  }
  
  return {
    fileType: generator.fileType,
    filePath: content.filePath,
    written: writeResult.written,
    merged: writeResult.mergeConflicts.length > 0,
    deferred: !writeResult.written && writeResult.deferred,
    contentHash: writeResult.contentHash,
    durationMs: performance.now() - startTime
  };
}
```

---

## Deferred Writes

When a generated file is open and unsaved in the VS Code editor, Roadie defers the write to avoid overwriting the developer's in-progress edits.

```tsx
class DeferredWriteQueue {
  private pending = new Map<string, GeneratedSection[]>();
  
  queue(filePath: string, sections: GeneratedSection[]): void {
    this.pending.set(filePath, sections);
    logger.info(`Deferred write for ${filePath} (file open in editor)`);
  }
  
  // Called when a file is saved in VS Code
  async onFileSaved(filePath: string): Promise<void> {
    if (this.pending.has(filePath)) {
      const sections = this.pending.get(filePath)!;
      this.pending.delete(filePath);
      
      // Now safe to write
      const existing = await readFile(filePath);
      await sectionManager.writeSectionFile(filePath, sections, existing);
      logger.info(`Deferred write completed for ${filePath}`);
    }
  }
}
```

---

## Parallel Generation

Generators are independent and can run in parallel:

```tsx
async function generateAll(): Promise<GenerationResult[]> {
  const generators = this.registeredGenerators;
  
  // Run all generators in parallel
  const results = await Promise.allSettled(
    generators.map(gen => runGenerationPipeline(gen, this.model))
  );
  
  // Collect results
  return results.map((result, i) => {
    if (result.status === 'fulfilled') return result.value;
    logger.error(`Generator ${generators[i].fileType} failed: ${result.reason}`);
    return {
      fileType: generators[i].fileType,
      filePath: '',
      written: false,
      merged: false,
      deferred: false,
      contentHash: '',
      durationMs: 0
    };
  });
}
```

---

## .gitignore Management

On first generation, ensure `.github/.roadie/.gitignore` exists:

```tsx
async function ensureGitignore(): Promise<void> {
  const gitignorePath = '.github/.roadie/.gitignore';
  if (!await fileExists(gitignorePath)) {
    await writeFile(gitignorePath, 'project-model.db\n*.db-journal\n');
  }
}
```

---

## Performance Budget

### Aggregate Budgets

| Operation | Budget |
| --- | --- |
| Section Manager merge (per file) | <100 ms |
| Total pipeline (model change → files written) | <2.5 s |
| All 8 generators (parallel) | <2.0 s wall-clock |

### Per-Generator Budgets

Each generator has an individual timeout enforced via `Promise.race` in `runGenerationPipeline()`. Budgets are chosen so that the worst-case **parallel** execution is ≤ 2000 ms (the bottleneck is the slowest individual generator, not the sum), while the worst-case **sequential** fallback is still bounded (sum = 1900 ms, well within the 2.5 s pipeline budget).

| # | Generator | File Produced | Timeout | Rationale |
| --- | --- | --- | --- | --- |
| 1 | Copilot Instructions | `.github/copilot-instructions.md` | **400 ms** | Full dependency graph walk + tech-stack serialization |
| 2 | AGENTS.md | `AGENTS.md` | **300 ms** | Agent role table + workflow index (smaller than Copilot) |
| 3 | Path Instructions | `.github/instructions/*.instructions.md` | **300 ms** | Per-directory markdown fan-out (glob + template) |
| 4 | Agent Definitions | `.github/agents/*.md` | **250 ms** | Static template + project name substitution |
| 5 | Skills | `.github/skills/*.md` | **200 ms** | One-off template per workflow type |
| 6 | Hooks | `.github/hooks/*.sh` | **150 ms** | Small shell snippets, conditional on detected linter/formatter |
| 7 | Workflows | `.github/workflows/*.yml` | **150 ms** | YAML template (only regenerated on CI structure change) |
| 8 | Templates | `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/*.md` | **150 ms** | Static markdown |
| 9 | Codebase Dictionary | `.github/codebase-dictionary.md` | **300 ms** | SQLite read of all entities + markdown table rendering |
| — | **Sum (serial)** | — | **1900 ms** | Worst-case if parallel execution degrades |

⚠️ Generator 9 (Codebase Dictionary) fires on workflow_complete, not on model change events. It is NOT included in the 2 s model-change budget. It runs as a separate pipeline after each workflow completes.
| — | **Max (parallel)** | — | **400 ms** | The slowest single generator dictates the wall-clock floor |

Enforce with timing assertions in integration tests. Exceeding a per-generator budget produces a `GENERATOR_TIMEOUT` error (in the `RoadieError` taxonomy) and the corresponding `GenerationResult.written` is set to `false`; other generators proceed unaffected because they run in `Promise.allSettled`.

### Budget Enforcement Test

```tsx
it('enforces per-generator budgets individually', async () => {
  const budgets: Record<GeneratedFileType, number> = {
    copilot_instructions: 400,
    agents_md:            300,
    path_instructions:    300,
    agent_definitions:    250,
    skills:               200,
    hooks:                150,
    workflows:            150,
    templates:            150,
    codebase_dictionary:  300,
  };

  for (const [fileType, budget] of Object.entries(budgets)) {
    const slowGenerator = makeGeneratorThatTakes(budget + 50);
    manager.register(slowGenerator);
    const result = await manager.generate(fileType as GeneratedFileType);
    expect(result.written).toBe(false);
    expect(result.error?.code).toBe('GENERATOR_TIMEOUT');
  }
});

it('total wall-clock stays under 2000 ms in parallel mode', async () => {
  const manager = createTestManager();
  const start = performance.now();
  await manager.generateAll();
  expect(performance.now() - start).toBeLessThan(2000);
});
```

---

## Testing Strategy

```tsx
describe('FileGeneratorManager', () => {
  it('triggers correct generators on dependency change', async () => {
    const manager = createTestManager();
    const spies = spyOnGenerators(manager);
    
    await manager.onModelChanged({ techStack: [newEntry] });
    
    expect(spies.copilotInstructions).toHaveBeenCalled();
    expect(spies.agentsMd).toHaveBeenCalled();
    expect(spies.pathInstructions).not.toHaveBeenCalled();
  });
  
  it('runs generators in parallel', async () => {
    const manager = createTestManager();
    const startTime = performance.now();
    
    await manager.generateAll();
    
    const elapsed = performance.now() - startTime;
    // 8 generators at 200ms each sequentially = 1600ms
    // In parallel should be ~200ms
    expect(elapsed).toBeLessThan(500);
  });
  
  it('defers write when file is open in editor', async () => {
    const manager = createTestManager();
    mockFileOpenInEditor('.github/copilot-instructions.md');
    
    const results = await manager.generateAll();
    
    const copilotResult = results.find(r => r.fileType === 'copilot_instructions');
    expect(copilotResult?.deferred).toBe(true);
  });
  
  it('skips write when content unchanged', async () => {
    const manager = createTestManager();
    
    // Generate once
    await manager.generateAll();
    // Generate again with same model
    const results = await manager.generateAll();
    
    // All should be skipped (hash match)
    expect(results.every(r => !r.written)).toBe(true);
  });
  
  it('all generators complete within 2s', async () => {
    const manager = createTestManager();
    const startTime = performance.now();
    
    await manager.generateAll();
    
    expect(performance.now() - startTime).toBeLessThan(2000);
  });
});
```

---

## Build Prompt for AI Agent

```
Build the File Generator Manager module (M19) according to this spec.

This module orchestrates file generation. It does NOT generate content
itself — it calls generator sub-modules and manages the write pipeline.

Key requirements:
1. Subscribe to PersistentProjectModel.onModelChanged() events
2. Map model change types to affected generators
3. Run generators in parallel (Promise.allSettled)
4. Pass generated content through Section Manager for merge/write
5. Defer writes when files are open in VS Code editor
6. Log generation events to Learning Database
7. Ensure .github/.roadie/.gitignore exists
8. All async I/O (no blocking operations)
9. Total generation time < 2s
10. Include 15+ unit tests

Files to create:
- src/generator/file-generator-manager.ts (~200 lines)
- src/generator/deferred-write-queue.ts (~60 lines)
- src/generator/file-generator-manager.test.ts (~200 lines)

Do NOT create the individual generator sub-modules yet.
Create stub implementations that return empty content.
The actual generators will be built in subsequent modules.
```