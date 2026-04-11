# 📊 Module Build Order & Verification

# Module Build Order & Verification

## Exact Sequence for Implementing Phase 1 Modules

Build modules in this order. Each milestone has dependencies that must be complete before the next can begin.

---

## Build Sequence

### **Step 1: Foundation Types** (Milestone 0)

**Module:** `src/types.ts`  

**Dependencies:** None  

**Estimated Time:** 1 hour  

**Build Prompt:**

> Create `src/types.ts` by transcribing **all** TypeScript interfaces from the local file `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` (in this documentation repository). That file is the single source of truth for all module contracts. Include: the `IntentType` union, `ClassificationResult`, `IntentClassifier`, all `WorkflowState` / `WorkflowDefinition` / `WorkflowStep` / `WorkflowContext` / `StepResult` / `WorkflowResult` interfaces, all `AgentRole` / `ModelTier` / `ToolScope` / `AgentConfig` / `AgentResult` / `ToolCallResult` types, all `ProjectModel` / `TechStackEntry` / `DirectoryNode` / `DetectedPattern` / `ProjectCommand` / `DeveloperPreferences` / `ProjectContext` / `ProjectModelDelta` interfaces, `GeneratedFileType` / `GeneratedFile`, and `RoadieError` / `StepExecutionError`. Add JSDoc comments verbatim from that file. No implementation logic — types only. **Do NOT reference any external URL or Notion page.**
> 

**Verification:**

```bash
npm run build # Should succeed
npm run lint # No errors
```

**Definition of Done:**

- [ ]  All 50+ types exported from `types.ts`
- [ ]  All types have JSDoc comments
- [ ]  No circular imports
- [ ]  File builds cleanly

---

### **Step 2: Project Scaffolding** (Milestone 0)

**Modules:**

- `src/extension.ts` (activate/deactivate entry points)
- `src/container.ts` (dependency injection container)
- `src/chat-participant.ts` (Chat Participant registration, echo handler)
- `src/status-bar.ts` (status bar item)

**Dependencies:** types.ts  

**Estimated Time:** 2 hours  

**Build Prompt:**

> Create a VS Code extension skeleton for Roadie. `extension.ts` should have activate/deactivate entry points using vscode.commands.registerCommand. Create `container.ts` as an empty DI container (will populate later). `chat-participant.ts` should register @roadie Chat Participant and echo input back for now. `status-bar.ts` should create a status bar item showing "Roadie active". Update `package.json` with chatParticipants contribution point, activation events, and manifest configuration.
> 

**Verification:**

```bash
npm run build
npm run lint
F5 in VS Code # Open Extension Development Host
# Check: @roadie appears in chat dropdown
# Check: Typing "hello" echoes "hello" back
# Check: Status bar shows "Roadie active"
```

**Definition of Done:**

- [ ]  Extension Development Host launches with F5
- [ ]  @roadie appears in chat agent dropdown
- [ ]  Message to @roadie is echoed back
- [ ]  Status bar displays correctly
- [ ]  No console errors

---

### **Step 3: Mock Infrastructure** (Milestone 1)

**Modules:**

- `test/mocks/mock-language-model.ts` (mock LLM API)
- `test/mocks/mock-chat-response-stream.ts` (mock chat stream)
- `test/fixtures/` (canned responses for testing)

**Dependencies:** types.ts

**Estimated Time:** 1.5 hours

**Build Prompt:**

> Create `test/mocks/mock-language-model.ts` by transcribing the class definition below **verbatim**. Use a plain class (not `vi.fn()`) because downstream tests assert on `calls` field shape directly via array indexing. Do NOT rename fields, change method signatures, or reorder constructor args.
>
> Then create `test/fixtures/` containing the four JSON files below each in their own file, with **exactly** the field names and value types shown.
>
> Finally create `test/mocks/mock-chat-response-stream.ts` using the interface described after the fixtures.

```ts
// test/mocks/mock-language-model.ts

import type * as vscode from 'vscode';

/** Shape of each recorded call — field order is canonical; tests index by field name */
export interface MockCall {
  prompt: string;        // the full text of the user message
  modelFamily: string;   // e.g. 'copilot-gpt-4o', 'copilot-gpt-3.5-turbo'
  tools: string[];       // tool names passed in the request (empty array if none)
  timestamp: number;     // Date.now() at the time send() was called
}

export type MockMode = 'success' | 'throw' | 'timeout' | 'partial';

export interface MockLanguageModelOptions {
  response?: string;       // text returned on 'success' (default: 'Mock LLM response')
  error?: Error;           // error thrown on 'throw' (default: new Error('Mock LLM error'))
  delayMs?: number;        // artificial delay before response (default: 0)
  mode?: MockMode;         // behaviour mode (default: 'success')
}

export class MockLanguageModelChat implements vscode.LanguageModelChat {
  // Public identifier fields required by vscode.LanguageModelChat
  readonly id: string = 'mock-model';
  readonly name: string = 'Mock Language Model';
  readonly vendor: string = 'mock';
  readonly family: string = 'copilot-gpt-4o';
  readonly version: string = '1.0.0';
  readonly maxInputTokens: number = 128_000;

  // Recorded calls — assertions use calls[0].prompt, calls[0].modelFamily, etc.
  calls: MockCall[] = [];

  private response: string;
  private error: Error;
  private delayMs: number;
  private mode: MockMode;

  constructor(opts: MockLanguageModelOptions = {}) {
    this.response = opts.response ?? 'Mock LLM response';
    this.error    = opts.error   ?? new Error('Mock LLM error');
    this.delayMs  = opts.delayMs ?? 0;
    this.mode     = opts.mode    ?? 'success';
  }

  async sendRequest(
    messages: vscode.LanguageModelChatMessage[],
    options: vscode.LanguageModelChatRequestOptions,
    token: vscode.CancellationToken,
  ): Promise<vscode.LanguageModelChatResponse> {
    // Record the call
    const userMessage = messages.find(m => m.role === vscode.LanguageModelChatMessageRole.User);
    this.calls.push({
      prompt:      String(userMessage?.content ?? ''),
      modelFamily: this.family,
      tools:       (options.tools ?? []).map(t => t.name),
      timestamp:   Date.now(),
    });

    if (this.delayMs > 0) {
      await new Promise(resolve => setTimeout(resolve, this.delayMs));
    }

    if (token.isCancellationRequested) {
      throw new Error('Cancelled');
    }

    switch (this.mode) {
      case 'throw':
        throw this.error;

      case 'timeout':
        // Simulate a hung request — never resolves
        await new Promise<never>(() => undefined);
        throw new Error('unreachable'); // TypeScript narrowing

      case 'partial': {
        // Returns a stream that emits half the response then stops
        const half = this.response.slice(0, Math.floor(this.response.length / 2));
        return { stream: MockLanguageModelChat._streamOf(half), text: Promise.resolve(half) };
      }

      default: {
        // 'success'
        const r = this.response;
        return { stream: MockLanguageModelChat._streamOf(r), text: Promise.resolve(r) };
      }
    }
  }

  /** Reset recorded calls between tests */
  reset(): void {
    this.calls = [];
  }

  private static async *_streamOf(text: string): AsyncIterable<vscode.LanguageModelTextPart> {
    yield { value: text } as vscode.LanguageModelTextPart;
  }

  // Satisfy vscode.LanguageModelChat — not used in tests
  countTokens(
    _text: string | vscode.LanguageModelChatMessage,
    _token?: vscode.CancellationToken,
  ): Thenable<number> {
    return Promise.resolve(100);
  }
}
```

---

**Fixture: `test/fixtures/diagnostic-response.json`**

```json
{
  "intent": "bug_fix",
  "step": "diagnose",
  "errorLocation": {
    "file": "src/auth/login.ts",
    "line": 42,
    "column": 18
  },
  "rootCause": "user property accessed on undefined — fetchUser() returns undefined when session cookie is expired",
  "affectedSymbols": ["fetchUser", "LoginHandler.handle"],
  "confidence": 0.92,
  "suggestedFix": "Add null-check before accessing user.id on line 42"
}
```

**Fixture: `test/fixtures/code-fix-response.json`**

```json
{
  "intent": "bug_fix",
  "step": "generate_fix",
  "targetFile": "src/auth/login.ts",
  "startLine": 40,
  "endLine": 45,
  "originalCode": "const user = await fetchUser(token);\nconsole.log(user.id);",
  "fixedCode": "const user = await fetchUser(token);\nif (!user) {\n  throw new RoadieError({ code: 'USER_NOT_FOUND', category: 'validation', userFacing: true, message: 'Session expired. Please log in again.' });\n}\nconsole.log(user.id);",
  "explanation": "Added null-guard after fetchUser() to handle expired sessions gracefully"
}
```

**Fixture: `test/fixtures/feature-plan-response.json`**

```json
{
  "intent": "feature",
  "step": "plan",
  "featureName": "dark-mode-toggle",
  "filesToCreate": [
    { "path": "src/ui/theme-toggle.ts", "role": "ThemeToggle command handler" }
  ],
  "filesToModify": [
    { "path": "src/extension.ts",        "change": "register ThemeToggle command in activate()" },
    { "path": "package.json",            "change": "add roadie.toggleTheme command contribution" }
  ],
  "estimatedSteps": 3,
  "confidence": 0.88
}
```

**Fixture: `test/fixtures/review-findings-response.json`**

```json
{
  "intent": "review",
  "step": "findings",
  "severity": "medium",
  "findings": [
    {
      "id": "R001",
      "file": "src/engine/workflow-engine.ts",
      "line": 78,
      "category": "error-handling",
      "description": "Promise rejection not caught — unhandled rejection will crash extension host",
      "suggestion": "Add .catch() or wrap in try/catch"
    },
    {
      "id": "R002",
      "file": "src/classifier/local-classifier.ts",
      "line": 31,
      "category": "performance",
      "description": "RegExp compiled inside loop — move to module scope for <10ms target",
      "suggestion": "Hoist regex constants to file top-level"
    }
  ],
  "summary": "2 findings: 0 critical, 1 medium, 1 low",
  "approveForMerge": false
}
```

---

**`test/mocks/mock-chat-response-stream.ts`** — implement `vscode.ChatResponseStream`:

```ts
// test/mocks/mock-chat-response-stream.ts

import type * as vscode from 'vscode';

export interface CapturedOutput {
  type: 'text' | 'button' | 'reference' | 'filepaths';
  content: unknown;
}

export class MockChatResponseStream implements vscode.ChatResponseStream {
  captured: CapturedOutput[] = [];

  markdown(value: string | vscode.MarkdownString): void {
    this.captured.push({ type: 'text', content: typeof value === 'string' ? value : value.value });
  }

  button(command: vscode.Command): void {
    this.captured.push({ type: 'button', content: command });
  }

  reference(value: vscode.Uri | vscode.Location): void {
    this.captured.push({ type: 'reference', content: value });
  }

  filepaths(value: vscode.ChatResponseFileTree[]): void {
    this.captured.push({ type: 'filepaths', content: value });
  }

  // Satisfy remaining vscode.ChatResponseStream interface — unused in tests
  anchor(_value: vscode.Uri | vscode.Location, _title?: string): void { /* noop */ }
  progress(_value: string): void { /* noop */ }
  push(_part: vscode.ChatResponsePart): void { /* noop */ }

  reset(): void {
    this.captured = [];
  }

  /** Convenience: get all text output concatenated */
  get text(): string {
    return this.captured.filter(c => c.type === 'text').map(c => c.content as string).join('');
  }
}
```

**Verification:**

```bash
npm run test
```

**Definition of Done:**

- [ ] `class MockLanguageModelChat` exists in `test/mocks/mock-language-model.ts` — exact shape above
- [ ] `MockCall` interface has fields **in this exact order**: `prompt`, `modelFamily`, `tools`, `timestamp`
- [ ] All 4 fixture JSON files exist in `test/fixtures/` with the exact field names shown above
- [ ] `MockLanguageModelChat` supports all 4 modes: `success`, `throw`, `timeout`, `partial`
- [ ] `reset()` clears `calls[]` — downstream `beforeEach()` hooks call this
- [ ] `MockChatResponseStream.captured` records all `button()` calls (needed by HITL tests)



---

### **Step 4: Model Resolver** (Milestone 1)

**Module:** `src/engine/model-resolver.ts`  

**Dependencies:** types.ts, mock infrastructure  

**Estimated Time:** 1.5 hours  

**Build Prompt:**

> Create ModelResolver that maps model tiers (free/standard/premium) to available models via vscode.lm.selectChatModels(). Implement: `resolve(tier: ModelTier): LanguageModelChat` returns the best available model for the tier. If the tier has no available models, fall back to lower tier. Add unit tests for each tier mapping and fallback behavior. Use mocks to simulate different model availability.
> 

**Verification:**

```bash
npm run test src/engine/model-resolver.test.ts
```

**Definition of Done:**

- [ ]  free tier maps to available models (e.g. GPT-4.1 or GPT-5 mini — actual names resolved at runtime via `vscode.lm.selectChatModels()`)
- [ ]  standard tier maps to available models (e.g. Claude Sonnet or GPT-5.2)
- [ ]  premium tier maps to available models (e.g. Claude Opus)
- [ ]  Fallback logic works (standard → free if standard unavailable)
- [ ]  8+ unit tests

---

### **Step 5: Intent Classifier** (Milestone 2)

**Modules:**

- `src/classifier/intent-patterns.ts` (pattern map)
- `src/classifier/intent-classifier.ts` (classifier logic)

**Dependencies:** types.ts, model-resolver, mocks  

**Estimated Time:** 3 hours  

**Build Prompt:**

> Implement two-tier intent classification. `intent-patterns.ts` contains a map of regex/keyword patterns to intent types with weights. `intent-classifier.ts` has three methods: `classify(prompt: string): ClassificationResult` for local classification (no I/O), `getClassificationPromptPrefix(): string` which returns the structured-output prefix to prepend to the system prompt when LLM classification is needed, and `parseClassification(responseText: string): ClassificationResult | null` which extracts the classification JSON from the first LLM response. This is the DOUBLE-DUTY pattern: classification piggybacks on the first LLM response rather than making a separate call. Do NOT create a `classifyWithLLM` method. Local classifier computes confidence from weighted signals. Set requiresLLM=true if confidence < 0.7.
>
> **CRITICAL — source of truth for patterns:** Copy `INTENT_PATTERNS` and `CONFIDENCE_THRESHOLDS` verbatim from `06_Workflows_and_Prompts/Intent Classification Taxonomy.md`. Do NOT invent pattern weights or threshold values. The exact TypeScript constants are inlined in that file under the "Canonical Pattern Weight Matrix" section. Use them as-is.
>
> Include 100+ test cases covering all 8 intent types and edge cases.



**Verification:**

```bash
npm run test src/classifier/
# Should achieve ≥90% accuracy on test cases
```

**Definition of Done:**

- [ ]  Local classifier achieves ≥90% accuracy on 100+ test cases
- [ ]  All 8 intents detectable
- [ ]  Ambiguous prompts produce confidence ≤0.6
- [ ]  Negative signals correctly reduce confidence
- [ ]  LLM fallback: `getClassificationPromptPrefix()` returns valid prefix
- [ ]  LLM fallback: `parseClassification()` extracts intent from response
- [ ]  No `classifyWithLLM` method exists (uses double-duty pattern)
- [ ]  <10ms latency for local classifier

---

### **Step 6: Workflow Engine Core** (Milestone 3)

**Modules:**

- `src/engine/step-executor.ts` (step execution + retry/escalation)
- `src/engine/workflow-engine.ts` (state machine + orchestration)
- `src/engine/definitions/` (workflow definition files - stub for now)

**Dependencies:** types.ts, model-resolver, mocks  

**Estimated Time:** 4 hours  

**Build Prompt:**

> Implement the workflow state machine. `workflow-engine.ts` accepts a WorkflowDefinition and WorkflowContext, executes steps sequentially, streams progress to chatResponseStream, manages state transitions (PENDING → RUNNING → RETRYING → COMPLETED/PAUSED/FAILED/CANCELLED), respects cancellationToken. `step-executor.ts` executes individual steps with a subagent, implements retry/escalation: attempt 1 (same tier), attempt 2 (refined prompt), attempt 3 (higher tier), attempt 4-6 (report and pause). Enforce step timeout. Include tests for sequential execution, retry logic, escalation, cancellation, and timeout handling.
> 

**Verification:**

```bash
npm run test src/engine/workflow-engine.test.ts
npm run test src/engine/step-executor.test.ts
```

**Definition of Done:**

- [ ]  4-step sequential workflow completes with mocks
- [ ]  Step results passed to next step
- [ ]  Retry: step fails attempt 1, succeeds attempt 2 with refined prompt
- [ ]  Escalation: step fails free tier, succeeds standard tier
- [ ]  3 consecutive failures → PAUSED state
- [ ]  Cancellation stops at step boundary
- [ ]  Timeout enforced
- [ ]  15+ unit tests

---

### **Step 7: Agent Spawner + Prompt Builder** (Milestone 4)

**Modules:**

- `src/spawner/prompt-builder.ts` (three-layer prompt construction)
- `src/spawner/tool-registry.ts` (tool scoping)
- `src/spawner/agent-spawner.ts` (subagent creation)

**Dependencies:** types.ts, model-resolver, mocks  

**Estimated Time:** 3 hours  

**Build Prompt:**

> Implement AgentSpawner. `prompt-builder.ts` constructs three-layer prompts: (1) role prompt, (2) context injection, (3) task prompt. `tool-registry.ts` maintains all tools and returns scoped subsets per step type. `agent-spawner.ts` accepts AgentConfig, uses model resolver for model selection, constructs prompt, scopes tools, sends request via vscode.lm, returns AgentResult. Implement `spawnParallel()` with Promise.allSettled() for concurrent agents. Include tests for prompt construction, tool scoping, and parallel execution.
> 

**Verification:**

```bash
npm run test src/spawner/
```

**Definition of Done:**

- [ ]  Spawned agent receives correct three-layer prompt
- [ ]  Research step has read-only tools
- [ ]  Implementation step has read/write tools
- [ ]  spawnParallel: 3 agents run concurrently
- [ ]  One failing parallel branch doesn't block others
- [ ]  Token usage tracked
- [ ]  10+ unit tests

---

### **Step 8: Project Model & Database** (Milestone 5)

**Modules:**

- `src/model/database.ts` (SQLite via better-sqlite3)
- `src/model/project-model.ts` (in-memory model + debounced writes)
- `src/analyzer/dependency-scanner.ts` (reads package.json, lock files)
- `src/analyzer/directory-scanner.ts` (fast-glob scanning)
- `src/analyzer/project-analyzer.ts` (orchestrates analysis)

**Dependencies:** types.ts  

**Estimated Time:** 5 hours  

**Build Prompt:**

> Implement lazy project model. `database.ts` creates SQLite at .github/.roadie/project-model.db, handles schema migrations, CRUD for tech stack/dependencies/directories. `project-model.ts` is in-memory with debounced SQLite writes (every 5s). Implement getTechStack(), getDirectoryStructure(), toContext() (serializes for LLM). `dependency-scanner.ts` reads package.json, lock files (pnpm-lock.yaml, yarn.lock), detects package manager, versions. `directory-scanner.ts` uses fast-glob to scan directories, assigns roles (source/test/config/output). `project-analyzer.ts` orchestrates analysis with plugin architecture (only Node.js plugin for v1.0). Include tests on fixture Node.js projects.
> 

**Verification:**

```bash
npm run test src/model/
npm run test src/analyzer/
# Against fixture projects in test/fixtures/node-js/
```

**Definition of Done:**

- [ ]  Analyzing a Next.js + Prisma project detects: TypeScript, Next.js, Prisma, pnpm
- [ ]  SQLite database created at correct path
- [ ]  Model loads from SQLite on re-activation
- [ ]  toContext() produces readable serialized output
- [ ]  Directory scanner ignores node_modules, .git, dist, build, coverage
- [ ]  12+ unit tests

---

### **Step 9: Bug Fix Workflow** (Milestone 6 - First End-to-End)

**Modules:**

- `src/engine/definitions/bug-fix.ts` (workflow definition)
- Updated `src/shell/chat-participant.ts` (route intent to workflows)

**Dependencies:** All previous modules  

**Estimated Time:** 4 hours  

**Build Prompt:**

> Implement the bug-fix workflow. Define `bug-fix.ts` as a WorkflowDefinition with 8 sequential steps: (1) locate error source, (2) diagnose, (3) generate fix, (4) verify with tests, (5) scan siblings, (6) fix siblings, (7) add regression test, (8) summary. Each step has a prompt template (use role prompts from Workflow Definitions page), tool scope, and model tier. Step 4 invokes shell command for test runner. Escalation: if step 4 fails, retry step 3 with test output + higher tier. Wire ChatParticipantHandler to route bug_fix intent to workflow engine. Manual test in Extension Development Host with a real Node.js project.
> 

**Verification:**

```bash
# Manual test: Extension Dev Host
# Type to @roadie: "The login page throws a 500 error"
# Observe: Workflow executes 8 steps, shows progress, delivers result
```

**Definition of Done:**

- [ ]  Bug-fix workflow definition complete (8 steps)
- [ ]  Chat Participant routes bug_fix → workflow engine
- [ ]  Progress streamed to chat ("Locating...", "Diagnosing...", etc.)
- [ ]  Step 4 (tests) runs project test command
- [ ]  Escalation on test failure works
- [ ]  Manual test succeeds

---

### **Step 10: Basic File Generation** (Milestone 7)

**Modules:**

- `src/generator/file-generator.ts` (orchestrates generation)
- `src/generator/section-manager.ts` (ownership markers, hashing)
- `src/generator/templates/copilot-instructions.ts` (template)
- `src/generator/templates/agent-definitions.ts` (template)

**Dependencies:** types.ts, project-model  

**Estimated Time:** 2 hours  

**Build Prompt:**

> Implement file generation. `file-generator.ts` reads project model and generates files using TypeScript template strings. `section-manager.ts` manages ownership markers (<!-- roadie:start:section --> / <!-- roadie:end:section -->), computes SHA-256 hashes, compares against existing files. Before writing, hash-compare new content — if identical, skip write. Create `.github/.roadie/.gitignore` with project-model.db. Generate `.github/copilot-instructions.md` (tech stack, package manager, commands) and `AGENTS.md` (project overview). Include tests verifying file content, markers, and no-write-if-identical logic.
> 

**Verification:**

```bash
npm run test src/generator/
# After bug-fix workflow, check files appear as unstaged changes
```

**Definition of Done:**

- [ ]  [copilot-instructions.md](http://copilot-instructions.md) generated after workflow
- [ ]  [AGENTS.md](http://AGENTS.md) generated
- [ ]  Files contain section markers
- [ ]  Regeneration with identical content doesn't touch files
- [ ]  .github/.roadie/.gitignore exists
- [ ]  8+ tests

---

### **Step 11: Remaining Workflows** (Milestones 8-12)

**Modules:**

- `src/engine/definitions/feature.ts` (feature development)
- `src/engine/definitions/refactor.ts` (refactoring)
- `src/engine/definitions/review.ts` (code review)
- `src/engine/definitions/document.ts` (documentation)
- `src/engine/definitions/dependency.ts` (dependency management)
- `src/engine/definitions/onboard.ts` (onboarding)

**Dependencies:** All engine modules  

**Estimated Time:** 8 hours total  

**Build Prompt (per workflow):**

> [Feature] Implement feature-development workflow (7 steps): analyze requirements → present plan (with approval buttons) → parallel layer agents (database, backend, frontend) → integrate → tests → quality review → commit messages. Step 2 uses stream.button() for Approve/Revise. Step 3 spawns 3 agents in parallel with Promise.allSettled().
> 

> [Refactor] Implement refactoring workflow (5 steps with inner loop): analyze → write characterization tests → [refactor → verify tests]* (loop) → summary. Key: public API invariant — stop if refactoring changes public interface.
> 

> [Review] Implement 5-pass code review (all parallel): security (Tier 1), performance, quality, test coverage, standards (all Tier 0). Consolidate findings. Review targets git diff.
> 

> [Remaining three] Similar detailed workflows from the Workflow Definitions page.
> 

**Verification:**

```bash
Manual test each workflow in Extension Dev Host
```

**Definition of Done (per workflow):**

- [ ]  Workflow definition complete
- [ ]  All steps have prompt templates
- [ ]  Correct model tiers assigned
- [ ]  Parallel/sequential/conditional logic correct
- [ ]  Manual test succeeds
- [ ]  5+ step tests

---

### **Step 12: Configuration & Polish** (Milestone 13)

**Modules:**

- Updated `src/extension.ts` (read configuration)
- `src/shell/commands.ts` (command palette registrations)

**Dependencies:** All modules  

**Estimated Time:** 1.5 hours  

**Build Prompt:**

> Add configuration support. Read `roadie.*` settings from `vscode.workspace.getConfiguration()`: testTimeout (default 300s), modelPreference (economy/balanced/quality), telemetry (default false), autoCommit (default false). Register command-palette commands: `roadie.init`, `roadie.rescan`, `roadie.reset`. Update package.json with configuration schema. Add settings tests.
> 

**Verification:**

```bash
npm run test src/shell/commands.test.ts
# Manually: change testTimeout to 10, trigger test step, verify timeout
```

**Definition of Done:**

- [ ]  All configuration options working
- [ ]  Commands registered and callable
- [ ]  Configuration tests pass
- [ ]  marketplace assets created (icon, README, CHANGELOG)

---

### **Step 13: Marketplace Preparation & Publishing** (Milestone 14)

**Action:** Package and publish to VS Code Marketplace  

**Dependencies:** All prior steps  

**Estimated Time:** 1 hour  

**Action:**

1. `vsce package` — generates .vsix file
2. Create marketplace listing (description, screenshots, categories)
3. Submit to marketplace as Preview/Pre-release
4. Publish

**Verification:**

```bash
vsce package
# Install .vsix in fresh VS Code
# Verify all features work
```

**Definition of Done:**

- [ ]  .vsix file generated
- [ ]  Published to marketplace as Preview
- [ ]  Installation from marketplace works
- [ ]  All workflows functional out-of-box

---

## Critical Path & Dependency Graph

```
Step 1 (types) →
Step 2 (scaffold) + Steps 3-7 (in parallel: mocks, engine, spawner)
  → Step 8 (project model) →
Step 9 (bug-fix workflow) [FIRST END-TO-END] →
Step 10 (file gen) →
Step 11 (remaining workflows) →
Step 12 (config) →
Step 13 (publish)
```

**Critical Path:** Steps 1, 2, 8, 9, 13 are on critical path. Parallelize Steps 3-7 for speed.

---

## Module Count & Milestones

| Phase | Modules | Status | Est. Time |
| --- | --- | --- | --- |
| M0 (Scaffold) | 4 | Foundation | 3h |
| M1-M4 (Engine) | 6 | Core Logic | 11h |
| M5 (Model) | 4 | Lazy State | 5h |
| M6-M12 (Workflows) | 7 + updates | Feature Complete | 12h |
| M13-14 (Polish + Publish) | 2 | Marketplace | 2.5h |
| **TOTAL** | **14 modules** | **Phase 1 Complete** | **~33.5 hours** |

---

**Next:** Go to Module Specifications to see detailed spec for each module.