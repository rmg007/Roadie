# 📐 Implementation Patterns & Standards

## Rules for AI Agents Implementing Phase 1 Modules

---

## Code Style & Structure

### File Length Constraint

- **Maximum 300 lines per file** (including tests in separate files)
- If a module design exceeds 300 lines, split into logical sub-modules
- **Example:** If `workflow-engine.ts` would be 400 lines, split into `workflow-engine.ts` (engine logic, ~200 lines) + `step-executor.ts` (step execution, ~200 lines) + move transitions to `workflow-states.ts` (~100 lines)

### Module Header

Every module must start with a JSDoc block:

```tsx
/**
 * @module intent-classifier
 * @description Two-tier intent classification for chat prompts.
 *   Tier 1: Local keyword/regex matching (instant, zero cost).
 *   Tier 2: LLM-based classification via structured output.
 * @inputs Developer chat prompt (string)
 * @outputs ClassificationResult (intent, confidence, signals)
 * @depends-on intent-patterns.ts (pattern map)
 * @depended-on-by chat-participant.ts (shell layer)
 */
```

### Imports Organization

```tsx
// 1. External dependencies
import * as vscode from 'vscode';
import { z } from 'zod';

// 2. Shared types
import { ClassificationResult, IntentType } from './types';

// 3. Internal dependencies (from same project)
import { IntentPatterns } from './intent-patterns';
```

---

## Type Safety & Validation

### Input Validation with Zod

All inputs crossing module boundaries must be validated with Zod schemas:

```tsx
import { z } from 'zod';

// Define schema at module level
const ClassifyInputSchema = z.object({
  prompt: z.string().min(1).max(10000),
  context: z.record(z.unknown()).optional(),
});

// Use in function
function classify(input: unknown): ClassificationResult {
  const validated = ClassifyInputSchema.parse(input);
  // ... process validated input
}
```

### Export Type Contracts

Every exported function must have explicit type annotations:

```tsx
// ✅ Good: Explicit input and output types
export async function executeWorkflow(
  definition: WorkflowDefinition,
  context: WorkflowContext
): Promise<WorkflowResult> {
  // ...
}

// ❌ Bad: Missing type annotations
export async function executeWorkflow(definition, context) {
  // ...
}
```

---

## Error Handling

### Throw or Return, Never Silently Fail

Every error must be either:

1. **Thrown** with a descriptive message, OR
2. **Returned** as a typed `Result<Success, Error>` (for recoverable errors)

```tsx
// ✅ Good: Throw for unexpected/unrecoverable
try {
  const result = await runTests();
  if (!result.success) {
    throw new Error(`Tests failed: ${result.stderr}`);
  }
} catch (error) {
  // Re-throw with context
  throw new Error(
    `Step 4 (verify fix) failed: ${error.message}`
  );
}

// ✅ Good: Return result for expected failures
function validateFixSyntax(code: string): Result<void, SyntaxError> {
  try {
    // validate
    return { ok: true };
  } catch (error) {
    return { ok: false, error: error as SyntaxError };
  }
}

// ❌ Bad: Silent failure
function validateFixSyntax(code: string): boolean {
  try {
    // validate
    return true;
  } catch (error) {
    return false; // Loss of context!
  }
}
```

### Structured Error Information

```tsx
class StepExecutionError extends Error {
  constructor(
    public stepId: string,
    public attempt: number,
    public maxRetries: number,
    public lastOutput: string,
    message: string
  ) {
    super(message);
    this.name = 'StepExecutionError';
  }
}

// Usage
throw new StepExecutionError(
  'step-3',
  3,
  3,
  testOutput,
  'Step 3 failed after 3 attempts. Last error: '
);
```

---

## Async & Concurrency

### Always Use async/await, Never Raw Promises

```tsx
// ✅ Good
async function processSteps(steps: WorkflowStep[]): Promise<StepResult[]> {
  const results = [];
  for (const step of steps) {
    const result = await executeStep(step);
    results.push(result);
  }
  return results;
}

// ❌ Bad: Raw callbacks
function processSteps(steps, callback) {
  let completed = 0;
  steps.forEach(step => {
    executeStep(step, (result) => {
      completed++;
      if (completed === steps.length) callback(results);
    });
  });
}
```

### Parallel Execution with Promise.allSettled

```tsx
// ✅ Good: All branches complete, no fail-fast
const results = await Promise.allSettled(
  config.branches.map(branch => executeStep(branch))
);

// Process results
const successes = results
  .filter((r) => r.status === 'fulfilled')
  .map((r) => (r as PromiseFulfilledResult<StepResult>).value);

const failures = results
  .filter((r) => r.status === 'rejected')
  .map((r) => (r as PromiseRejectedResult).reason);
```

### Timeout Enforcement

```tsx
// ✅ Good: Explicit timeout with Promise.race
async function executeWithTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number
): Promise<T> {
  const timeoutPromise = new Promise<T>((_, reject) =>
    setTimeout(
      () => reject(new Error(`Operation timed out after ${timeoutMs}ms`)),
      timeoutMs
    )
  );
  return Promise.race([promise, timeoutPromise]);
}
```

---

## Testing

### Test Co-Location

```
src/
├── intent-classifier.ts      (module)
├── intent-classifier.test.ts (tests for module)
├── workflow-engine.ts        (module)
└── workflow-engine.test.ts   (tests for module)
```

### Test Structure with Vitest

```tsx
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { IntentClassifier } from './intent-classifier';

describe('IntentClassifier', () => {
  let classifier: IntentClassifier;

  beforeEach(() => {
    classifier = new IntentClassifier();
  });

  describe('classify', () => {
    it('should detect bug_fix intent from "fix the login error"', () => {
      const result = classifier.classify('fix the login error');
      expect(result.intent).toBe('bug_fix');
      expect(result.confidence).toBeGreaterThan(0.7);
    });

    it('should fallback to general_chat when no intent matches', () => {
      const result = classifier.classify('what is the capital of france');
      expect(result.intent).toBe('general_chat');
    });

    it('should set requiresLLM=true when confidence < 0.7', () => {
      const result = classifier.classify('update dependencies please');
      if (result.confidence < 0.7) {
        expect(result.requiresLLM).toBe(true);
      }
    });
  });
});
```

### Mock LLM Classification (Double-Duty Pattern)

```tsx
// The classifier uses the double-duty pattern:
// 1. classify() runs locally
// 2. If requiresLLM, ChatParticipantHandler prepends getClassificationPromptPrefix()
// 3. After LLM responds, parseClassification() extracts the intent

// In test setup
const mockResponse = '{"intent": "bug_fix", "reasoning": "User mentions error"}\n\nI can help fix that...';

// Test parseClassification
const result = classifier.parseClassification(mockResponse);
expect(result?.intent).toBe('bug_fix');
expect(result?.confidence).toBeGreaterThanOrEqual(0.85);

// Test getClassificationPromptPrefix
const prefix = classifier.getClassificationPromptPrefix();
expect(prefix).toContain('bug_fix');
expect(prefix).toContain('JSON');
```

---

## VS Code API Usage

### Window/UI Interactions

```tsx
// ✅ Good: Use UI for user-facing errors
if (testsFailed) {
  await vscode.window.showErrorMessage(
    `Tests failed: ${testOutput}`,
    'View Logs'
  );
}

// ✅ Good: Output channel for non-blocking logs
const outputChannel = vscode.window.createOutputChannel('Roadie');
outputChannel.appendLine(`Step 3: Generating fix...`);
```

### Configuration Access

```tsx
// ✅ Good: Read configuration once and cache
const config = vscode.workspace.getConfiguration('roadie');
const testTimeout = config.get<number>('testTimeout', 300000);

// Listen for changes
const disposable = vscode.workspace.onDidChangeConfiguration((e) => {
  if (e.affectsConfiguration('roadie.testTimeout')) {
    // Re-read config
  }
});
```

---

## LLM Interaction Patterns

### Prompt Construction

```tsx
// ✅ Good: Structured, three-part prompts
const systemPrompt = `
You are a ${role}. Your job is to: ${task}

Project Context:
${projectContext}

Constraints:
- Do not modify public APIs
- Only use tools listed below
- Report findings, don't execute
`;

const userPrompt = `
${developerMessage}

Previous step result:
${previousOutput}
`;
```

### Structured Output Parsing

```tsx
// ✅ Good: Parse structured output with validation
const response = await model.sendRequest({
  // ... prompt ...
});

const jsonMatch = response.text.match(/\{[\s\S]*\}/);
if (!jsonMatch) throw new Error('No JSON found in response');

const parsed = JSON.parse(jsonMatch[0]);
const validated = ClassificationSchema.parse(parsed);
// Now use validated
```

---

## Performance & Limits

### Respect Budgets

From TAD:

| Operation | Budget | Check |
| --- | --- | --- |
| File watcher debounce | 500ms | Use debounce helper |
| Intent classification | <10ms | No I/O in local classifier |
| Project model flush | Every 5s max | Use debounced timer |
| Dependency file parse | <200ms | Timeout wrapper |
| Workflow step timeout | 60s default | Use Promise.race |

### Per-Generator Timeout Budget

The project scanner runs multiple generators in sequence. The aggregate scan time target is **<2000ms** per run. Individual generator timeout budgets:

| Generator | File Glob | Timeout (ms) | Notes |
|---|---|---|---|
| `PackageJsonGenerator` | `package.json` | 150 | Top-level + workspaces |
| `TechStackGenerator` | `**/*.json`, `**/*.toml`, `**/*.gradle` | 400 | Multi-file, deepest scan |
| `TestCommandGenerator` | `package.json`, `pyproject.toml`, `Makefile`, `go.mod` | 150 | See detection algorithm below |
| `DirectoryRoleGenerator` | `**/` (dirs only) | 300 | Skips `node_modules`, `.git` |
| `PatternDetectorGenerator` | `src/**/*.ts`, `lib/**/*.ts` | 300 | Sample up to 100 files |
| `GitContextGenerator` | `.git/HEAD`, `.git/config` | 100 | Stat-only; read-on-demand |
| `ConfigFileGenerator` | `tsconfig.json`, `*.config.*` | 150 | Shallow, config files only |
| `LanguageRatioGenerator` | `src/**/*` | 350 | Line count sampling |
| **Total** | | **≤1900ms** | 100ms buffer to 2000ms limit |

**Enforcement:** Each generator wraps its glob + parse logic in `executeWithTimeout(promise, <budget>)`. If a generator times out, it logs a warning and returns an empty result — other generators are NOT cancelled.

---

### Test Command Detection Algorithm

When `roadie.testCommand` is empty (default), Roadie detects the test command using this decision tree. Implementation lives in `src/generators/test-command-generator.ts`.

```
1. READ {workspaceRoot}/package.json
   IF exists:
     CHECK scripts.test — if non-empty and not "echo", use "npm test"
     CHECK scripts["test:unit"] — prefer over bare "test" if present
     DETECT package manager:
       IF pnpm-lock.yaml exists → "pnpm test"
       ELSE IF yarn.lock exists  → "yarn test"
       ELSE                      → "npm test"
     RETURN detected command (confidence: 0.95)

2. READ {workspaceRoot}/pyproject.toml
   IF exists:
     CHECK [tool.pytest.ini_options] section — if present → "pytest"
     CHECK [tool.poetry.dev-dependencies] for pytest — if found → "pytest"
     RETURN "pytest" (confidence: 0.90)

3. READ {workspaceRoot}/Cargo.toml
   IF exists:
     CHECK [package] section — always has cargo test
     RETURN "cargo test" (confidence: 0.99)

4. READ {workspaceRoot}/go.mod
   IF exists:
     RETURN "go test ./..." (confidence: 0.99)

5. READ {workspaceRoot}/Makefile
   IF exists:
     GREP for "^test:" target — if found, use "make test"
     RETURN "make test" (confidence: 0.80)

6. NO test command detected
   RETURN null (confidence: 0.0)
   LOG warning: "[TestCommandGenerator] No test command detected. Set roadie.testCommand in settings."
```

**TypeScript shape:**
```ts
interface TestCommandResult {
  command: string | null;
  confidence: number;  // 0.0 – 1.0
  source: 'package.json' | 'pyproject.toml' | 'Cargo.toml' | 'go.mod' | 'Makefile' | 'undetected';
}
```

---

## Section Manager Concurrency Contract

`SectionManager` can receive concurrent write requests from two sources: the generator pipeline (scheduled scan) and the edit tracker (user saves a generated file). Without explicit serialization, these produces interleaved writes and data loss.

### Deferred-Write Queue

```ts
// src/generators/section-manager.ts

class SectionManager {
  private _writeQueue: Array<() => Promise<void>> = [];
  private _writing = false;

  async writeSection(filePath: string, sectionId: string, content: string): Promise<void> {
    return new Promise((resolve, reject) => {
      this._writeQueue.push(async () => {
        try {
          await this._writeSectionWithMtimeGuard(filePath, sectionId, content);
          resolve();
        } catch (err) {
          reject(err);
        }
      });
      this._drainQueue();
    });
  }

  private async _drainQueue(): Promise<void> {
    if (this._writing) return;
    this._writing = true;
    while (this._writeQueue.length > 0) {
      const task = this._writeQueue.shift()!;
      await task();
    }
    this._writing = false;
  }

  private async _writeSectionWithMtimeGuard(
    filePath: string,
    sectionId: string,
    content: string,
  ): Promise<void> {
    // 1. Stat the file and record mtime BEFORE reading
    const statBefore = await fs.promises.stat(filePath).catch(() => null);
    const mtimeBefore = statBefore?.mtimeMs ?? 0;

    // 2. Read the current file content
    const existing = await fs.promises.readFile(filePath, 'utf8').catch(() => '');

    // 3. Stat AGAIN — if mtime changed between stat and read, another process wrote
    const statAfter = await fs.promises.stat(filePath).catch(() => null);
    const mtimeAfter = statAfter?.mtimeMs ?? 0;

    if (mtimeAfter !== mtimeBefore) {
      // File was modified concurrently — re-queue and retry once
      logger.warn(`[SectionManager] mtime drift on ${filePath} — re-queuing write`);
      this._writeQueue.unshift(() => this._writeSectionWithMtimeGuard(filePath, sectionId, content));
      return;
    }

    // 4. Merge section into existing content and write
    const merged = mergeSection(existing, sectionId, content);
    await fs.promises.writeFile(filePath, merged, 'utf8');
  }
}
```

**Test case requirement:**
```ts
it('handles concurrent generator + edit-tracker writes without data loss', async () => {
  const sectionManager = new SectionManager();
  // Trigger two concurrent writeSection calls to the same file
  await Promise.all([
    sectionManager.writeSection('CLAUDE.md', 'tech-stack', 'Content A'),
    sectionManager.writeSection('CLAUDE.md', 'commands',   'Content B'),
  ]);
  const result = await fs.promises.readFile('CLAUDE.md', 'utf8');
  expect(result).toContain('Content A');
  expect(result).toContain('Content B');
});
```

---

## File Watcher Symlink Handling

The file watcher uses `vscode.workspace.createFileSystemWatcher()`. On monorepos with `packages/*` symlinks, a change to a symlinked file produces **two** events (one for the symlink path, one for the real path), causing duplicate scan runs.

### Resolution: fs.realpath Deduplication

```ts
// src/watchers/file-watcher.ts

import { realpath } from 'fs/promises';

class FileWatcher {
  private _pendingPaths = new Set<string>();

  async onFileChanged(uri: vscode.Uri): Promise<void> {
    let realFsPath: string;
    try {
      // Resolve symlink to canonical path before processing
      realFsPath = await realpath(uri.fsPath);
    } catch {
      // realpath fails if the file was deleted — use the original path
      realFsPath = uri.fsPath;
    }

    // Deduplicate: if the real path is already in the pending set, skip
    if (this._pendingPaths.has(realFsPath)) {
      return;
    }
    this._pendingPaths.add(realFsPath);

    // Debounce: process after 500ms quiet period
    setTimeout(() => {
      this._pendingPaths.delete(realFsPath);
      this._processChange(realFsPath);
    }, 500);
  }

  private _processChange(realPath: string): void {
    // Trigger re-scan for the affected generator
    this._projectAnalyzer.scanPath(realPath);
  }
}
```

**Polling fallback:** If the VS Code file system watcher does not emit events for a path (e.g., on network drives or certain WSL mounts), Roadie falls back to polling at 5-second intervals. Trigger condition: `vscode.workspace.createFileSystemWatcher` emits zero events in the first 30 seconds after activation in a workspace where files are known to exist.

---

### Memory Limits

```tsx
// ✅ Good: Cap directory tree depth
const MAX_TREE_DEPTH = 10;
const MAX_SAMPLES = 100; // Pattern detection

function scanDirectory(path: string, depth = 0): DirectoryNode {
  if (depth > MAX_TREE_DEPTH) {
    return { path, type: 'directory', children: [] }; // Skip deep paths
  }
  // ...
}
```

---


## Logging

### Structured Logging

```tsx
// ✅ Good: Structured, contextual logs
const logger = {
  info: (msg: string, context?: Record<string, unknown>) =>
    console.log(`[INFO] ${msg}`, context),
  error: (msg: string, error?: Error) =>
    console.error(`[ERROR] ${msg}`, error?.message),
  debug: (msg: string, context?: Record<string, unknown>) =>
    process.env.DEBUG && console.log(`[DEBUG] ${msg}`, context),
};

// Usage
logger.info('Step 3: Executing fix', { stepId: 'step-3', modelTier: 'standard' });
```

---

## What to AVOID

❌ **Circular imports** — Always import downward (shell → engine → model)  

❌ **Global state** — Pass context explicitly  

❌ **Silent failures** — Always throw or return error  

❌ **Raw callbacks** — Always use async/await  

❌ **Unvalidated input** — Always use Zod  

❌ **Commented-out code** — Delete or ticket it  

❌ **TODOs without context** — Link to issues or be specific  

❌ **Magic numbers** — Define as named constants  

❌ **Type assertions (`as`)** — Use proper validation instead  

❌ **console.log in production code** — Use logger or vscode.OutputChannel

❌ **Synchronous file I/O in Phase 1.5 code** — All watcher/generator paths MUST use `fs.promises` (async). `fs.readFileSync`/`fs.writeFileSync` block the extension host.  

---

## Quick Checklist Before Submitting Code

- [ ]  All functions have type annotations
- [ ]  All cross-module inputs validated with Zod
- [ ]  No errors are silently swallowed
- [ ]  Tests exist and pass
- [ ]  No file exceeds 300 lines
- [ ]  Module header JSDoc is present
- [ ]  No circular imports
- [ ]  Performance budgets respected
- [ ]  Uses async/await, never raw callbacks
- [ ]  Handles cancellation tokens (if applicable)

---

**Next:** Go to Module Build Order to start implementing.