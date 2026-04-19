# Roadie — Technical Architecture Document

**CONFIDENTIAL** · Version 1.0 — April 2026

Platform: VS Code Extension (TypeScript) | Runtime: Node.js 20.x+ | Database: SQLite

---

# 1. System Architecture

## 1.1 Architecture Overview

Roadie operates as a VS Code extension with two concurrent modes: Active Mode (developer-initiated chat workflows) and Passive Mode (automated file watching and generation). Both modes share a unified project model backed by SQLite.

### 1.1.1 Active Mode Data Flow

Developer sends a chat message to Roadie via the VS Code Chat Participant. The intent classifier analyzes the prompt locally using keyword/regex matching. If confidence is high, the appropriate workflow is selected; otherwise, the first LLM call classifies the intent with structured output. The workflow engine executes a multi-step state machine, spawning ephemeral subagents for each step. Results stream back through the Chat Participant to the developer.

### 1.1.2 Passive Mode Data Flow

The VS Code FileSystemWatcher monitors the workspace for changes to dependency files, source code, and configuration. Changes trigger incremental updates to the in-memory project model, which are debounced and persisted to SQLite. When the project model changes, the file generator evaluates whether any .github/ files need updating and regenerates affected sections while respecting human-edited content.

## 1.2 Component Relationship Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  VS Code Extension Shell (Chat Participant + UI)            │
│    ┌───────────────┐  ┌─────────────┐  ┌─────────────┐     │
│    │ Sidebar View  │  │ Status Bar  │  │ File Watcher│     │
│    └───────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
         │ chat input               │ file events
         ▼                          ▼
  ┌─────────────────┐     ┌──────────────────┐
  │ Intent Classifier│     │ Project Analyzer │
  └────────┬────────┘     └────────┬─────────┘
         │ intent + confidence     │ updates
         ▼                          ▼
  ┌─────────────────┐     ┌──────────────────┐
  │ Workflow Engine  │─────│  Project Model   │───┐
  └────────┬────────┘     └──────────────────┘   │
         │ spawn subagents         │ reads        │
         ▼                          ▼              │
  ┌─────────────────┐     ┌──────────────────┐   │
  │  Agent Spawner  │     │  File Generator  │   │
  └────────┬────────┘     └────────┬─────────┘   │
         │ results                 │ writes       │
         ▼                          ▼              │
  ┌─────────────────────────────────────────┐   │
  │  Learning Database (SQLite)             │───┘
  │  .github/.roadie/project-model.db       │
  └─────────────────────────────────────────┘
```

## 1.3 Operating Modes Summary

| Aspect | Active Mode | Passive Mode |
| --- | --- | --- |
| Trigger | Developer chat message | File system event |
| Interaction | Conversational, streamed responses | Silent, background |
| Components | Intent Classifier → Workflow Engine → Agent Spawner | File Watcher → Project Analyzer → File Generator |
| State | Ephemeral (in-memory only) | Persistent (SQLite) |
| Phase | Phase 1 (launch) | Phase 1.5 |

---

# 2. Component Specifications

## 2.1 VS Code Extension Shell

**Responsibility:** Entry point for the extension. Registers the Chat Participant, sidebar webview, status bar item, file watcher, and commands. Manages lifecycle (activation, deactivation). Routes messages between VS Code APIs and the core engine modules.

### 2.1.1 Public Interface

```tsx
interface ExtensionShell {
  activate(context: vscode.ExtensionContext): Promise<void>;
  deactivate(): Promise<void>;
  getProjectModel(): ProjectModel;
  getWorkflowEngine(): WorkflowEngine;
  getLearningDatabase(): LearningDatabase;
}
```

### 2.1.2 Internal Structure

- `extension.ts` — activate/deactivate entry points, dependency injection container setup
- `chat-participant.ts` — Registers @roadie chat participant, handles `vscode.ChatRequestHandler`
- `sidebar-provider.ts` — WebviewViewProvider for project status sidebar *(deferred — not implemented in Phase 1/1.5)*
- `status-bar.ts` — StatusBarItem showing current workflow state
- `commands.ts` — Command palette registrations (roadie.init, roadie.rescan, roadie.reset)

### 2.1.3 Dependencies

vscode API (Chat Participant, Language Model API, WebviewView, StatusBarItem, FileSystemWatcher), all core engine modules via dependency injection.

### 2.1.4 Error Handling

All errors from core modules are caught at the shell boundary. User-facing errors are shown via `vscode.window.showErrorMessage()`. Non-critical errors (file watcher failures, background scan errors) are logged to the output channel but do not interrupt the developer.

## 2.2 Intent Classifier

**Responsibility:** Analyzes developer chat prompts and classifies them into workflow types. Two-tier system: local keyword/regex classifier runs first (instant, zero cost), and if confidence is below threshold, the first LLM call classifies with structured output.

### 2.2.1 Public Interface

```tsx
interface IntentClassifier {
  classify(prompt: string): ClassificationResult;
  parseClassification(responseText: string): ClassificationResult | null;
  getClassificationPromptPrefix(): string;
}

interface ClassificationResult {
  intent: IntentType;
  confidence: number; // 0.0 to 1.0
  signals: string[]; // matched keywords/patterns
  requiresLLM: boolean; // true if local confidence < 0.7
}

type IntentType = 'bug_fix' | 'feature' | 'refactor' | 'review'
  | 'document' | 'dependency' | 'onboard' | 'general_chat';
```

> **Revision note (TAD-6):** The interface uses `parseClassification()` and `getClassificationPromptPrefix()` instead of `classifyWithLLM()` to explicitly support the double-duty pattern where classification piggybacks on the first LLM response rather than making a separate call.
> 

### 2.2.2 Internal Structure

- `intent-classifier.ts` — Main module with `classify()`, `parseClassification()`, and `getClassificationPromptPrefix()`
- `intent-patterns.ts` — Map of regex/keyword patterns to intent types with weights

### 2.2.3 Error Handling

Local classifier never throws — returns `general_chat` with low confidence on any error. LLM classifier falls back to `general_chat` if the structured output parse fails.

## 2.3 Workflow Engine

**Responsibility:** State machine that executes multi-step workflows. Each workflow is a declarative definition (states, transitions, handlers). The engine is imperative. Supports sequential steps, parallel steps, conditional branching, retry with escalation, model switching per step, and timeout/cancellation.

### 2.3.1 Public Interface

```tsx
interface WorkflowEngine {
  execute(workflowId: string, context: WorkflowContext): Promise<WorkflowResult>;
  cancel(executionId: string): void;
  getActiveWorkflows(): WorkflowExecution[];
  registerWorkflow(definition: WorkflowDefinition): void;
}

interface WorkflowContext {
  prompt: string;
  intent: ClassificationResult;
  projectModel: ProjectModel;
  chatResponseStream: vscode.ChatResponseStream;
  cancellationToken: vscode.CancellationToken;
  previousStepResults?: StepResult[]; // accumulated results from prior steps
}
```

> **Revision note (TAD-3):** `previousStepResults` added to enable the bug-fix escalation pattern where step 4's test output must be passed back to step 3 as additional context.
> 

### 2.3.2 Internal Structure

- `workflow-engine.ts` — Core execution loop, state transitions, parallel orchestration
- `workflow-definitions/` — Directory of declarative workflow definitions (one file per workflow type)
- `step-executor.ts` — Executes individual steps, manages retry/escalation
- `model-resolver.ts` — Maps model tiers to available models at runtime

### 2.3.3 Error Handling

Three-tier escalation: (1) Retry same model with refined prompt. (2) Retry with more capable model following cost tier hierarchy. (3) Report to developer with summary of attempts. Workflow pauses on failed step but does not abort — independent steps continue.

## 2.4 Project Analyzer

**Responsibility:** Reads dependency files, scans directories, detects patterns. Builds and maintains the project model. Lazy evaluation — only analyzes what is needed. All results persisted to SQLite.

### 2.4.1 Public Interface

```tsx
interface ProjectAnalyzer {
  analyze(scope: AnalysisScope): Promise<AnalysisResult>;
  detectPatterns(fileTypes?: string[]): Promise<DetectedPattern[]>;
  scanDependencies(): Promise<DependencyInfo[]>;
  invalidate(paths: string[]): void;
}

type AnalysisScope = 'full' | 'dependencies' | 'patterns' | 'structure';
```

### 2.4.2 Supported Dependency Files

| Language/Platform | Dependency File | Lock File |
| --- | --- | --- |
| Node.js | package.json | package-lock.json / yarn.lock / pnpm-lock.yaml |
| Python | pyproject.toml / requirements.txt | poetry.lock / uv.lock |
| Go | go.mod | go.sum |
| Rust | Cargo.toml | Cargo.lock |
| Java | pom.xml / build.gradle | N/A |
| C# | *.csproj | packages.lock.json |
| Ruby | Gemfile | Gemfile.lock |

### 2.4.3 Pattern Detection Examples

- **Export style:** Named exports only (no default exports detected in N scanned files) — detected by regex scanning export statements.
- **Test convention:** Co-located test files using `{name}.test.ts` pattern in `src/` — detected by finding test files and analyzing naming/location.
- **Error handling:** Uses custom AppError class from `src/errors/` (found in N catch blocks) — detected by searching throw/catch patterns.
- **Import ordering:** External imports first, then `@/` aliases, then relative paths — detected by sampling import blocks.
- **Commit convention:** Conventional commits (feat/fix/chore prefixes in recent 50 commits) — detected via git log analysis.

All patterns are detected with regex/string analysis, not AST parsing.

## 2.5 Project Model

**Responsibility:** In-memory representation of project state. Loaded from SQLite at extension activation. Updated incrementally. Debounced writes to SQLite (batch every 5 seconds max). Survives restarts. Located at `.github/.roadie/project-model.db`.

### 2.5.1 Public Interface

```tsx
interface ProjectModel {
  getTechStack(): TechStack;
  getDirectoryStructure(): DirectoryNode;
  getPatterns(): DetectedPattern[];
  getPreferences(): DeveloperPreferences;
  getWorkflowHistory(): WorkflowHistoryEntry[];
  update(delta: ProjectModelDelta): void;
  toContext(options?: {
    maxTokens?: number;
    scope?: 'full' | 'stack' | 'structure' | 'commands' | 'patterns';
    relevantPaths?: string[];
  }): ProjectContext;
}
```

> **Revision note (TAD-2):** `toContext()` now accepts optional parameters for token budgeting, scope filtering, and path relevance. This allows the prompt builder to request only the context that fits the current step's token budget.
> 

### 2.5.2 Error Handling

If the SQLite database is corrupted or missing, the project model initializes empty and triggers a full re-scan. Database migrations are handled via a version table with sequential migration scripts.

## 2.6 Agent Spawner

**Responsibility:** Creates ephemeral subagents for workflow steps. Each subagent gets a specific role, scoped tool set, model selection, and focused prompt derived from workflow context. Uses the VS Code Language Model API.

### 2.6.1 Public Interface

```tsx
interface AgentSpawner {
  spawn(config: AgentConfig): Promise<AgentResult>;
  spawnParallel(configs: AgentConfig[]): Promise<AgentResult[]>;
}

interface AgentConfig {
  role: AgentRole;
  modelTier: 'free' | 'standard' | 'premium';
  tools: ToolScope;
  promptTemplate: string;
  context: Record<string, unknown>;
  timeoutMs: number;
}
```

### 2.6.2 Tool Scoping

| Step Type | Available Tools |
| --- | --- |
| Research / Diagnostic | Read file, search workspace, list directory, read terminal output |
| Implementation | Read file, edit file, create file, delete file, run terminal command |
| Review | Read file, run terminal command (tests/lint), search workspace |
| Documentation | Read file, search workspace, list directory |

## 2.7 File Generator

**Responsibility:** Produces Markdown files in `.github/` from the project model and workflow learnings. Respects section ownership (auto-generated vs human-written). Template-string-based generation.

### 2.7.1 Public Interface

```tsx
interface FileGenerator {
  generate(fileType: GeneratedFileType): Promise<GeneratedFile>;
  generateAll(): Promise<GeneratedFile[]>;
  detectHumanEdits(filePath: string): Promise<EditDetection>;
  merge(generated: string, existing: string): string;
}
```

### 2.7.2 Generated Files

- `.github/copilot-instructions.md` — Repo-wide Copilot instructions
- `.github/instructions/*.instructions.md` — Path-specific instructions per language/framework
- `.github/agents/*.agent.md` — Specialized agents based on workflow usage
- `.github/skills/*/SKILL.md` — Procedural skills from detected workflows
- `.github/hooks/*.json` — Lifecycle hooks (formatting, security gates)
- `.github/workflows/pr-test.yml` — PR test workflow (if none exists)
- `.github/PULL_REQUEST_TEMPLATE.md` — PR template
- `.github/ISSUE_TEMPLATE/bug_report.md` and `feature_request.md` — Issue templates
- `AGENTS.md` — Root-level cross-agent instructions

### 2.7.3 Section Ownership

Ownership is tracked via HTML comments in the Markdown files: `<!-- roadie:start:section-name -->` and `<!-- roadie:end:section-name -->`. A SHA-256 hash of the last generated content per section is stored in the learning database. If the current file content hash differs from the stored hash, the section was human-edited — Roadie appends new content below the existing content (separated by `<!-- roadie:merged:timestamp -->`), never overwriting.

## 2.8 Learning Database

**Responsibility:** SQLite storage for edit tracking (full text snapshots), workflow outcomes, discovered patterns, and developer preferences. Located at `.github/.roadie/project-model.db` (shared with project model).

### 2.8.1 Public Interface

```tsx
interface LearningDatabase {
  recordSnapshot(filePath: string, content: string, source: 'roadie' | 'human'): void;
  getSnapshots(filePath: string, limit?: number): FileSnapshot[];
  recordWorkflowOutcome(entry: WorkflowHistoryEntry): void;
  getWorkflowHistory(limit?: number): WorkflowHistoryEntry[];
  recordPattern(pattern: DetectedPattern): void;
  prune(): void;
}
```

### 2.8.2 Retention Policy

- Last 50 snapshots per file
- Last 100 workflow history entries total
- Pruning runs on extension activation (not during workflows)
- Target: SQLite file stays under 10 MB for typical projects

## 2.9 MCP Server (Phase 2)

**Responsibility:** Exposes core capabilities as MCP tools. Runs as stdio-based server spawned by the extension. Uses `@modelcontextprotocol/sdk`. Phase 2 — the TAD defines interface boundaries only.

### 2.9.1 Interface Boundary

The following core interfaces will be wrapped as MCP tools in Phase 2. The current architecture ensures these interfaces are clean and self-contained, so the MCP server becomes a thin wrapper rather than a refactor.

| Core Interface | Future MCP Tool Category | Description |
| --- | --- | --- |
| ProjectAnalyzer.analyze() | Project Info | Scan project structure, dependencies, patterns |
| ProjectModel.toContext() | Project Context | Retrieve serialized project context for LLM prompts |
| WorkflowEngine.execute() | Workflow Execution | Trigger and monitor workflows |
| FileGenerator.generate() | File Generation | Generate or update .github/ files |
| LearningDatabase.getWorkflowHistory() | Learning Data | Query workflow history and patterns |

MCP tool schemas, transport details, and server configuration are deferred to Phase 2. The design goal: if these interfaces are clean now, the MCP server is a thin wrapper later.

---

# 3. Data Architecture

## 3.1 SQLite Schema

All persistent data lives in a single SQLite database at `.github/.roadie/project-model.db`. The schema uses four major table groups: project model, file snapshots, workflow history, and detected patterns.

### 3.1.1 Project Model Tables

```sql
CREATE TABLE schema_version (
  version INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE tech_stack (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  category TEXT NOT NULL,
  name TEXT NOT NULL,
  version TEXT,
  source_file TEXT NOT NULL,
  detected_at TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(category, name)
);

CREATE TABLE directory_structure (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  path TEXT NOT NULL UNIQUE,
  type TEXT NOT NULL,
  language TEXT,
  last_scanned TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 3.1.2 File Snapshot Tables

```sql
CREATE TABLE file_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path TEXT NOT NULL,
  content TEXT NOT NULL,
  content_hash TEXT NOT NULL,
  source TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_snapshots_path ON file_snapshots(file_path, created_at DESC);
```

### 3.1.3 Workflow History Tables

```sql
CREATE TABLE workflow_history (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  workflow_type TEXT NOT NULL,
  prompt TEXT NOT NULL,
  status TEXT NOT NULL,
  steps_completed INTEGER NOT NULL DEFAULT 0,
  steps_total INTEGER NOT NULL DEFAULT 0,
  duration_ms INTEGER,
  error_summary TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 3.1.4 Detected Patterns Table

```sql
CREATE TABLE detected_patterns (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  category TEXT NOT NULL,
  description TEXT NOT NULL,
  evidence TEXT NOT NULL,
  confidence REAL NOT NULL,
  detected_at TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(category)
);
```

### 3.1.5 Developer Preferences Table

```sql
CREATE TABLE developer_preferences (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 3.1.6 Codebase Dictionary Tables

```sql
CREATE TABLE IF NOT EXISTS codebase_entities (
  id                  INTEGER PRIMARY KEY AUTOINCREMENT,
  name                TEXT    NOT NULL,
  kind                TEXT    NOT NULL CHECK(kind IN (
                        'function','class','interface','type','enum',
                        'constant','route','model','component')),
  file_path           TEXT    NOT NULL,
  line_number         INTEGER,
  signature           TEXT,
  purpose             TEXT    DEFAULT '',
  is_exported         INTEGER NOT NULL DEFAULT 1,
  created_by_workflow TEXT,
  created_at          TEXT    NOT NULL DEFAULT (datetime('now')),
  updated_at          TEXT    NOT NULL DEFAULT (datetime('now')),
  UNIQUE(file_path, name, kind)
);

CREATE INDEX IF NOT EXISTS idx_entities_file
  ON codebase_entities(file_path);
CREATE INDEX IF NOT EXISTS idx_entities_name
  ON codebase_entities(name);

CREATE TABLE IF NOT EXISTS entity_relationships (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  source_id    INTEGER NOT NULL REFERENCES codebase_entities(id) ON DELETE CASCADE,
  target_id    INTEGER NOT NULL REFERENCES codebase_entities(id) ON DELETE CASCADE,
  relationship TEXT    NOT NULL,
  UNIQUE(source_id, target_id, relationship)
);

CREATE TABLE IF NOT EXISTS entity_modifications (
  id                  INTEGER PRIMARY KEY AUTOINCREMENT,
  entity_id           INTEGER NOT NULL REFERENCES codebase_entities(id) ON DELETE CASCADE,
  workflow_type       TEXT,
  step_id             TEXT,
  original_prompt     TEXT,
  change_description  TEXT,
  modified_at         TEXT    NOT NULL DEFAULT (datetime('now'))
);
```

## 3.2 In-Memory Model Structure

```tsx
interface ProjectModelState {
  techStack: TechStackEntry[];
  directoryTree: DirectoryNode;
  patterns: DetectedPattern[];
  preferences: Map<string, unknown>;
  sectionHashes: Map<string, string>;
  isDirty: boolean;
  lastFlushedAt: number;
}
```

## 3.3 File System Layout

```
.github/
├── copilot-instructions.md
├── instructions/
│   ├── typescript.instructions.md
│   ├── python.instructions.md
│   └── react.instructions.md
├── agents/
│   ├── reviewer.agent.md
│   └── debugger.agent.md
├── skills/
│   └── deployment/
│       └── SKILL.md
├── hooks/
│   ├── pre-commit.json
│   └── security-scan.json
├── workflows/
│   └── pr-test.yml
├── PULL_REQUEST_TEMPLATE.md
├── ISSUE_TEMPLATE/
│   ├── bug_report.md
│   └── feature_request.md
└── .roadie/
    └── project-model.db
```

The `.roadie/` directory is added to `.gitignore` by default. All other generated files are committed to version control.

---

# 4. Workflow Engine Design

## 4.1 State Machine Model

Each workflow execution is a finite state machine. States are defined declaratively in workflow definition files. The engine traverses states based on transition rules, executing handler functions at each state.

### 4.1.1 Execution States

| State | Description |
| --- | --- |
| PENDING | Workflow created but not started |
| RUNNING | Currently executing a step |
| WAITING_PARALLEL | Waiting for all parallel branches to complete |
| RETRYING | A step failed and is being retried (with escalation) |
| PAUSED | Awaiting developer input after all retries exhausted |
| COMPLETED | All steps finished successfully |
| FAILED | Workflow aborted (only on cancellation or unrecoverable error) |
| CANCELLED | Developer cancelled the workflow |

## 4.2 Workflow Definition Schema

```tsx
interface WorkflowDefinition {
  id: string;
  name: string;
  steps: WorkflowStep[];
  onComplete?: (results: StepResult[]) => WorkflowResult;
}

interface WorkflowStep {
  id: string;
  name: string;
  type: 'sequential' | 'parallel' | 'conditional';
  agentRole: AgentRole;
  modelTier: 'free' | 'standard' | 'premium';
  toolScope: ToolScope;
  promptTemplate: string;
  timeoutMs: number;
  maxRetries: number; // default 3
  branches?: WorkflowStep[]; // Parallel sub-steps
  condition?: (context: WorkflowContext) => string; // returns next step ID
}
```

## 4.3 Step Execution Model

Each step is executed by the step executor, which spawns a subagent via the Agent Spawner. The executor manages timeouts, retries, and model escalation for each step independently.

### 4.3.1 Single Step Lifecycle

1. Step executor receives step definition and workflow context.
2. Model resolver maps the step's model tier to an available model via `vscode.lm.selectChatModels()`.
3. Agent spawner creates a subagent with the resolved model, scoped tools, and constructed prompt.
4. Subagent executes. Results stream back through the chat response stream.
5. On success, the result is stored and the engine transitions to the next state.
6. On failure, the retry/escalation logic is triggered.

## 4.4 Parallel Execution

Parallel steps use `Promise.allSettled()` to let all branches complete independently. No fail-fast. Failed branches are retried independently up to 3 attempts with model escalation. The aggregation step collects all results and notes any branches that failed after all retries. The developer sees a complete report with partial results rather than nothing.

```tsx
// Parallel execution pseudocode
const results = await Promise.allSettled(
  step.branches.map(branch => stepExecutor.execute(branch, context))
);
// Each rejected result triggers independent retry
```

## 4.5 Retry and Escalation

Escalation follows a three-tier strategy:

- **Tier 1 — Same model, refined prompt.** The step executor modifies the prompt to include the error from the previous attempt and asks the model to try a different approach.
- **Tier 2 — More capable model.** The model resolver selects the next higher tier: free → standard → premium. If the current tier has no alternative, skip to tier 3.
- **Tier 3 — Developer escalation.** After 3 failed attempts, the workflow engine reports to the developer via the chat response stream with a summary of what was attempted and why each attempt failed. The workflow pauses on that step. Other independent steps continue executing.

## 4.6 Cancellation

The workflow engine respects `vscode.CancellationToken` at every step boundary. When cancellation is requested, the engine sets the workflow state to CANCELLED, aborts any pending LLM requests, and records the partial results to the learning database. In-flight subagents are not forcibly killed but their results are discarded.

---

# 5. Intent Classification Design

## 5.1 Two-Tier Classification

Intent classification uses a two-tier approach. Tier 1 is a local keyword/regex classifier that runs instantly with zero cost. If Tier 1 confidence is below the threshold (0.7), Tier 2 uses the first LLM call to classify with structured output before proceeding with the workflow.

## 5.2 Intent Taxonomy

| Intent | Signal Words / Patterns | Example Prompts |
| --- | --- | --- |
| bug_fix | fix, bug, broken, error, not working, crash, failing, issue | "Fix the login error on the settings page" |
| feature | add, create, build, new feature, implement, make | "Add a dark mode toggle to the sidebar" |
| refactor | refactor, clean up, restructure, simplify, extract, optimize | "Refactor the auth module to use dependency injection" |
| review | review, check, audit, analyze, look at, evaluate | "Review my PR for security issues" |
| document | document, README, docs, explain, JSDoc, comments | "Document the API endpoints in this service" |
| dependency | update, upgrade, migrate, dependency, version, package | "Update React to v19" |
| onboard | onboard, new to, understand, how does, architecture | "Help me understand the project architecture" |
| general_chat | (fallback — no strong signals) | "What's a good pattern for error handling?" |

## 5.3 Confidence Scoring

The local classifier computes confidence based on the number and weight of matched signals. Primary signal words (e.g., "fix" for bug_fix) carry weight 0.4. Secondary signals (e.g., "error" for bug_fix) carry weight 0.2. Negative signals reduce confidence (e.g., "don't fix" reduces bug_fix confidence). If multiple intents score above 0.3, the highest wins but confidence is capped at 0.6 (ambiguous — triggers LLM classification).

## 5.4 LLM Classification (Tier 2)

When local confidence is below 0.7, the first LLM call includes a system prompt requesting structured output with the classification. The prompt asks the model to respond with a JSON block containing intent and reasoning before proceeding with the actual response. This avoids a separate classification call — one LLM call serves double duty.

```
// Structured output request appended to system prompt
// "Before responding, classify this request as one of:
// bug_fix | feature | refactor | review | document |
// dependency | onboard | general_chat
// Respond with: {"intent": "...", "reasoning": "..."}"
```

## 5.5 Fallback Behavior

The `general_chat` intent is the universal fallback. Unlike other intents, `general_chat` does not trigger a workflow — it enriches the prompt with project context from the project model (tech stack, directory structure, relevant file references) and sends it to the LLM as a single-turn request. The developer always benefits from Roadie being selected, even for casual questions.

---

# 6. Agent Spawning Design

## 6.1 Subagent Creation

Each workflow step spawns an ephemeral subagent via the Agent Spawner. The subagent is not a persistent entity — it exists only for the duration of a single step execution. The spawner constructs the subagent by combining the workflow step definition with the current workflow context.

## 6.2 Prompt Construction

Prompts are constructed from three layers:

- **Role prompt:** Defines the subagent's identity and constraints (e.g., "You are a diagnostic agent. Your job is to identify the root cause of the reported bug. Do not suggest fixes — only diagnose.").
- **Context injection:** Project model context (tech stack, relevant patterns, directory structure) is serialized and injected. Only context relevant to the current step is included to minimize token usage.
- **Task prompt:** The specific task derived from the workflow step template, with variables substituted from the workflow context (developer prompt, previous step results, file contents).

## 6.3 Model Selection

Workflow definitions specify model tiers per step type, not specific models. The model resolver maps tiers to available models at runtime using `vscode.lm.selectChatModels()`. This insulates workflow definitions from model catalog changes.

| Model Tier | Cost Factor | Typical Use | Fallback |
| --- | --- | --- | --- |
| free (0×) | Zero premium requests | Research, file reading, simple analysis | N/A (always available) |
| standard (1×) | 1 premium request per call | Implementation, code generation | Falls back to free |
| premium (3×) | 3 premium requests per call | Deep reasoning, complex review, architecture | Falls back to standard |

If a tier has no available models, the resolver falls back to the next lower tier. The developer can override tier-to-model mapping via extension settings.

## 6.4 Tool Scoping

The Agent Spawner maintains a global tool registry of all available VS Code Language Model API tools (`vscode.LanguageModelChatTool`). For each step, a subset of tools is selected based on the step type's tool scope definition. Tool scoping is defined per step type in the workflow definition, not per individual step.

## 6.5 Result Aggregation

Each subagent returns an `AgentResult` containing: the agent's text output, any tool call results (files modified, tests run, etc.), token usage metrics, and a status indicator. For parallel steps, results are collected via `Promise.allSettled()` and merged into a unified report. The aggregation preserves per-branch attribution so the developer can see which agent produced which findings.

---

# 7. File Generation Design

## 7.1 Template System

File generation uses TypeScript template strings, not a template engine. Each generated file type has a dedicated generator function that accepts the project model and returns the file content as a string. Template strings provide full TypeScript type safety and avoid the complexity of a template language.

```tsx
// Example: copilot-instructions.md generator
function generateCopilotInstructions(model: ProjectModel): string {
  const stack = model.getTechStack();
  return `<!-- roadie:start:tech-stack -->
## Tech Stack
${stack.map(s => `- ${s.name} ${s.version}`).join('\n')}
<!-- roadie:end:tech-stack -->`;
}
```

## 7.2 Section Ownership Markers

Each auto-managed section is delimited by HTML comments: `<!-- roadie:start:section-name -->` and `<!-- roadie:end:section-name -->`. Sections outside these markers are considered human-owned and are never modified by Roadie.

### 7.2.1 Human Edit Detection

When Roadie generates a section, it computes a SHA-256 hash of the content and stores it in the learning database. On the next generation cycle, Roadie reads the current file, extracts each section, and compares its hash to the stored hash. If they differ, the section was human-edited.

### 7.2.2 Merge Strategy

- **No human edits:** Replace the section content entirely with the new generated content.
- **Human edits detected:** Append the new generated content below the existing content within the same section markers, separated by a comment: `<!-- roadie:merged:timestamp -->`. The developer can then manually reconcile.
- **Section markers removed:** The entire file is considered human-owned. Roadie logs a warning and does not modify the file.

## 7.3 Update Diffing

Before writing any file, Roadie compares the new content to the existing file content. If they are identical (by hash comparison), no write occurs. This prevents unnecessary file system events and git noise.

## 7.4 Conflict Resolution

If a generated file has unsaved changes in the VS Code editor, Roadie defers the write until the file is saved or closed. This prevents overwriting the developer's in-progress edits. Deferred writes are queued and executed on the next file save event.

---

# 8. Extension Lifecycle

## 8.1 Activation

Roadie activates on the `onChat:roadie` activation event (when the developer first mentions @roadie in chat) or on `workspaceContains:.github/.roadie/project-model.db` (if Roadie was previously initialized in this workspace).

### 8.1.1 Activation Sequence

1. Load SQLite database from `.github/.roadie/project-model.db` (or create if missing).
2. Initialize in-memory project model from SQLite data.
3. Register Chat Participant (@roadie).
4. Register sidebar webview provider and status bar item.
5. Start VS Code FileSystemWatcher for dependency files and source directories.
6. Run background pruning of learning database (old snapshots, stale history).
7. Run lightweight project model validation (check if cached tech stack matches current dependency files).

## 8.2 Deactivation

On deactivation (VS Code closing, extension being disabled), Roadie flushes any pending project model writes to SQLite, disposes all file watchers, and cancels any active workflows. No attempt is made to persist in-flight workflow state.

## 8.3 State Persistence

| Data Type | Persisted? | Location | Survives Restart? |
| --- | --- | --- | --- |
| Project model | Yes | SQLite (.github/.roadie/project-model.db) | Yes |
| Learning data | Yes | SQLite (same database) | Yes |
| Workflow state | No | In-memory only | No |
| Chat history | No | Managed by VS Code | Managed by VS Code |
| Extension config | Yes | VS Code settings.json | Yes |

## 8.4 Crash Recovery

If VS Code crashes or the extension host terminates unexpectedly, the following recovery applies on next activation:

- SQLite database integrity is validated via `PRAGMA integrity_check`. If corrupt, the database is deleted and rebuilt from a fresh project scan.
- Any in-flight workflows are lost (ephemeral). The developer re-triggers by sending the same prompt.
- The file watcher re-initializes and catches up on any file changes that occurred while the extension was inactive.
- Debounced writes that were not flushed are lost, but the impact is minimal (at most 5 seconds of project model updates).

---

# 9. Security Model

## 9.1 Workspace Trust

Roadie respects VS Code's workspace trust model. In untrusted workspaces, Roadie operates in a restricted mode: file generation is disabled, workflow steps that execute terminal commands are blocked, and the project analyzer operates in read-only mode (no git operations).

## 9.2 Secret Handling

Roadie handles zero secrets. It relies entirely on the VS Code Language Model API for model access, which is authenticated via the developer's Copilot subscription. No API keys, tokens, or credentials are stored or managed by Roadie. The extension does not make any HTTP requests to external services.

## 9.3 Sensitive Content Exclusion

The primary security concern is ensuring generated files and workflow outputs do not accidentally include sensitive content from the project. Roadie excludes known sensitive patterns from all generated instruction content:

- `.env` file contents (values, not key names)
- Private keys and certificates (`*.pem`, `*.key` patterns)
- API keys and tokens (regex patterns for common key formats)
- Connection strings and database credentials
- Contents of directories named `secrets/`, `credentials/`, or `private/`

## 9.4 Tool Approval

All VS Code Language Model API tool calls go through the standard VS Code tool approval flow. The developer sees which tools a subagent wants to use and can approve or deny. Roadie does not bypass or auto-approve any tool calls.

## 9.5 Sandboxing

Subagents are sandboxed by tool scoping: a research agent cannot call edit tools, and a documentation agent cannot execute terminal commands. This is enforced at the Agent Spawner level by only passing the scoped tool set to the `sendRequest()` call. There is no additional OS-level sandboxing beyond what the VS Code extension host provides.

---

# 10. Module Boundaries

## 10.1 Module Map

The codebase is organized into discrete modules, each under 300 lines, optimized for AI agent productivity. Every module has a co-located test file and a JSDoc header block.

```
src/
├── extension.ts              // Entry point: activate/deactivate
├── types.ts                  // Shared interfaces and types
├── container.ts              // Dependency injection container
├── shell/
│   ├── chat-participant.ts   // Chat Participant handler
│   ├── sidebar-provider.ts   // Sidebar webview (deferred — not yet implemented)
│   ├── status-bar.ts         // Status bar item
│   └── commands.ts           // Command palette registrations
├── classifier/
│   ├── intent-classifier.ts  // Two-tier classification logic
│   └── intent-patterns.ts    // Pattern map for local classifier
├── engine/
│   ├── workflow-engine.ts    // State machine execution
│   ├── step-executor.ts      // Step execution + retry logic
│   ├── model-resolver.ts     // Tier-to-model resolution
│   └── definitions/          // One file per workflow type
│       ├── bug-fix.ts
│       ├── feature.ts
│       ├── refactor.ts
│       ├── review.ts
│       ├── document.ts
│       ├── dependency.ts
│       └── onboard.ts
├── analyzer/
│   ├── project-analyzer.ts   // Orchestrates analysis
│   ├── dependency-scanner.ts // Reads package.json, pyproject.toml, etc.
│   ├── pattern-detector.ts   // Regex-based pattern detection
│   └── directory-scanner.ts  // fast-glob directory scanning
├── model/
│   ├── project-model.ts      // In-memory model + debounced writes
│   └── database.ts           // SQLite operations (better-sqlite3)
├── spawner/
│   ├── agent-spawner.ts      // Subagent creation + parallel execution
│   ├── prompt-builder.ts     // Three-layer prompt construction
│   └── tool-registry.ts      // Global tool registry + scoping
├── generator/
│   ├── file-generator.ts     // Orchestrates file generation
│   ├── section-manager.ts    // Section ownership + hash tracking
│   └── templates/            // One file per generated file type
│       ├── copilot-instructions.ts
│       ├── path-instructions.ts
│       ├── agent-definitions.ts
│       ├── skill-definitions.ts
│       ├── hooks.ts
│       ├── pr-template.ts
│       └── issue-templates.ts
└── learning/
    └── learning-database.ts  // Snapshot storage + workflow history + pruning
```

## 10.2 Dependency Graph

Dependencies flow in one direction: shell → classifier/engine → spawner/analyzer → model/generator → learning. No circular dependencies. The `types.ts` file is the shared interface contract imported by all modules.

| Module | Depends On | Depended On By |
| --- | --- | --- |
| shell/* | classifier, engine, model | extension.ts |
| classifier/* | types | shell |
| engine/* | spawner, model, learning, types | shell |
| analyzer/* | model, types | engine, shell |
| model/* | types | engine, analyzer, generator, spawner |
| spawner/* | model, types | engine |
| generator/* | model, learning, types | engine, shell |
| learning/* | types | model, generator, engine |

> **Revision note (TAD-5):** The `model → learning` dependency from the original TAD has been removed. The project model depends on the analyzer for data, not the learning database. The learning database is written to by the engine and generator, and read by the generator for edit tracking.
> 

---

# 11. Testing Strategy

## 11.1 Test Framework

Unit tests use Vitest (fast, TypeScript-native, compatible with VS Code extension projects). Integration tests use `@vscode/test-cli` and `@vscode/test-electron` for end-to-end testing within a real VS Code instance.

## 11.2 Unit Test Coverage

| Module | What Is Tested | LLM Mock Strategy |
| --- | --- | --- |
| Intent Classifier | Pattern matching accuracy across 100+ prompt examples, confidence scoring, ambiguity detection | N/A (local classifier has no LLM) |
| Workflow Engine | State transitions, parallel execution, retry/escalation, cancellation, timeout handling | Mock AgentSpawner returns canned results |
| Model Resolver | Tier-to-model mapping, fallback behavior, config overrides | Mock vscode.lm.selectChatModels() |
| Project Analyzer | Dependency file parsing for all 7 ecosystems, pattern detection accuracy | N/A (no LLM calls) |
| Project Model | CRUD operations, debounced writes, SQLite serialization/deserialization | N/A |
| File Generator | Template output correctness, section ownership detection, merge logic, hash comparison | N/A |
| Learning Database | Snapshot storage/retrieval, pruning, retention limits | N/A |
| Section Manager | Hash computation, human edit detection, marker parsing | N/A |

## 11.3 Integration Test Coverage

- End-to-end chat flow: send a prompt via the Chat Participant, verify intent classification, workflow execution, and streamed response.
- File watcher: modify a dependency file, verify project model update and file regeneration.
- Workspace activation: open a workspace with an existing .roadie database, verify project model loads correctly.
- Crash recovery: corrupt the SQLite database, verify the extension recovers gracefully.

## 11.4 LLM Call Mocking

All LLM interactions go through the Agent Spawner, which uses the VS Code Language Model API. For testing, the `AgentSpawner` interface is mocked at the boundary. Test fixtures provide canned `AgentResult` objects for each workflow step type. This approach avoids mocking the VS Code API internals and keeps tests focused on business logic.

## 11.5 Test Data Management

- Fixture projects: a set of minimal test projects (Node.js, Python, Go, Rust, Java, C#, Ruby) stored in `test/fixtures/` with known dependency files and code patterns.
- Snapshot files: expected generated file outputs stored as `.snap` files alongside tests.
- SQLite test databases: pre-populated databases for testing migration and data access.
- Tests are co-located: `{module}.ts` has `{module}.test.ts` in the same directory.

---

# 12. Performance Budgets

## 12.1 Timing Budgets

| Operation | Budget | Enforcement |
| --- | --- | --- |
| File watcher debounce | 500ms | VS Code FileSystemWatcher with custom debounce wrapper |
| Intent classification (local) | <10ms | Monitored via [performance.now](http://performance.now)(); no I/O allowed |
| Project model SQLite flush | Every 5s max | Debounced timer; batch writes |
| Single dependency file parse | <200ms | Timeout wrapper; skip and log on exceed |
| Full project scan (initial) | <30s for 10K files | fast-glob with concurrency limit; progress reported |
| Pattern detection (per pattern) | <500ms | Sampling cap: max 100 files per pattern scan |
| Workflow step timeout | 60s default | Configurable per step in workflow definition |
| File generation (single file) | <1s | Template string execution is synchronous and fast |
| SQLite database pruning | <5s | Runs on activation only, not during workflows |

## 12.2 Memory Budgets

| Component | Memory Limit | Enforcement |
| --- | --- | --- |
| In-memory project model | <50 MB | Directory tree depth capped at 10 levels; large files excluded |
| Active workflow context | <20 MB per workflow | Token budgeting via @vscode/prompt-tsx |
| File watcher | <10 MB | VS Code FileSystemWatcher with ignored globs for node_modules, .git, dist |
| SQLite database file | <10 MB | Pruning on activation; retention limits enforced |
| Total extension memory | <100 MB | Monitored via process.memoryUsage(); warning logged at 80% |

## 12.3 File System Budgets

- Maximum scanned files: 10,000 (configurable). Projects exceeding this limit trigger a warning and use sampling-based analysis.
- Maximum generated files: 20 (current file list has 12; buffer for growth).
- Maximum file watcher paths: 500. Directories beyond this limit are excluded with a logged warning.
- Ignored paths (always): `node_modules`, `.git`, `dist`, `build`, `out`, `coverage`, `__pycache__`, `.next`, `.nuxt`, `vendor`.

---

# 13. Code Conventions for AI Agent Maintenance

All code in the Roadie codebase will be written and maintained by AI agents (Claude Code, Codex, or Roadie itself). The following conventions ensure any AI agent can understand and modify the codebase reliably.

## 13.1 File Length

Maximum file length: 300 lines. If a module exceeds this, it must be split into focused sub-modules. This constraint ensures each file fits comfortably within an AI agent's context window and reduces the scope of changes.

## 13.2 Module Header

Every module starts with a JSDoc block explaining its purpose, inputs/outputs, and relationship to other modules:

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

## 13.3 Test Co-location

Tests are co-located with their source: `{module}.ts` has `{module}.test.ts` in the same directory. This ensures an AI agent modifying a module always sees the corresponding tests in the same directory listing.

## 13.4 Interface-First Design

All module boundaries are defined by TypeScript interfaces in the shared `types.ts` file. Implementations import their interface contract from `types.ts`. This allows AI agents to understand module interactions by reading a single file.

## 13.5 No Circular Dependencies

Dependencies flow in one direction: shell → classifier/engine → spawner/analyzer → model/generator → learning. Circular dependencies are a build error enforced by the tsup bundler configuration.

## 13.6 [AGENTS.md](http://AGENTS.md)

An `AGENTS.md` file at the repository root describes the project architecture, module map, and key design decisions for AI agents. This file is the entry point for any AI agent working on the codebase and is kept in sync with the TAD.

## 13.7 Validation Schemas

All data crossing module boundaries is validated with Zod schemas. This provides runtime type safety and clear error messages when an AI agent produces incorrectly structured data during development.