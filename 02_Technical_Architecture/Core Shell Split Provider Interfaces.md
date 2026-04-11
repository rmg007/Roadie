# ✂️ Core/Shell Split — Provider Interfaces

## Provider Interfaces & Module Refactoring

---

## 1. The Problem

Several Phase 1/1.5 modules import from `vscode` directly. The MCP server must reuse these modules without the VS Code API.

| Module | `vscode` Dependency | Why |
| --- | --- | --- |
| `engine/model-resolver.ts` | `vscode.lm.selectChatModels()` | Model discovery |
| `spawner/agent-spawner.ts` | `vscode.lm.sendChatRequest()` | LLM calls |
| `engine/workflow-engine.ts` | `vscode.CancellationToken`, `vscode.ChatResponseStream` | Cancellation + streaming |
| `engine/step-executor.ts` | `vscode.CancellationToken` | Cancellation |
| `generator/file-generator.ts` | `vscode.workspace.openTextDocument()` | Deferred write check |
| `spawner/tool-registry.ts` | `vscode.LanguageModelChatTool` | Tool definitions |
| `shell/chat-participant.ts` | `vscode.ChatRequestHandler` | Chat UI (not shared) |
| `shell/status-bar.ts` | `vscode.StatusBarItem` | Status UI (not shared) |

**Modules that do NOT need changes** (already VS Code–independent):

- `classifier/intent-classifier.ts`
- `analyzer/project-analyzer.ts`
- `model/project-model.ts`
- `model/database.ts`
- `generator/section-manager.ts`
- `learning/learning-database.ts`
- All generator templates

## 2. The Solution: Provider Interfaces

**New file: `src/providers.ts`**

This file defines abstract interfaces that replace direct VS Code API usage. Each interface has two implementations: one for VS Code (in the extension shell), one for standalone (in the MCP server).

### ModelProvider

Abstracts `vscode.lm.selectChatModels()` + `vscode.lm.sendChatRequest()`.

```tsx
export interface ModelProvider {
  selectModels(selector: ModelSelector): Promise<ModelInfo[]>;
  sendRequest(
    modelId: string,
    messages: ChatMessage[],
    options: ModelRequestOptions
  ): Promise<ModelResponse>;
}

export interface ModelSelector {
  vendor?: string;
  family?: string;
  id?: string;
}

export interface ModelInfo {
  id: string;
  name: string;
  vendor: string;
  family: string;
  maxInputTokens: number;
}

export interface ChatMessage {
  role: 'system' | 'user' | 'assistant';
  content: string;
}

export interface ModelRequestOptions {
  tools?: ToolDefinition[];
  cancellation?: AbortSignal;
  modelOptions?: Record<string, unknown>;
}

export interface ModelResponse {
  text: string;
  toolCalls: ToolCallResult[];
  usage: { inputTokens: number; outputTokens: number };
}
```

### ProgressReporter

Abstracts `vscode.ChatResponseStream`.

```tsx
export interface ProgressReporter {
  report(message: string): void;
  reportMarkdown(markdown: string): void;
}
```

### CancellationHandle

Abstracts `vscode.CancellationToken`.

```tsx
export interface CancellationHandle {
  readonly isCancelled: boolean;
  onCancelled(callback: () => void): void;
}
```

### FileSystemProvider

Abstracts file system operations that may differ between VS Code and standalone.

```tsx
export interface FileSystemProvider {
  isFileOpenInEditor(filePath: string): boolean;
  readFile(filePath: string): Promise<string>;
  writeFile(filePath: string, content: string): Promise<void>;
  fileExists(filePath: string): Promise<boolean>;
}
```

### ConfigProvider

Abstracts VS Code configuration.

```tsx
export interface ConfigProvider {
  get<T>(key: string, defaultValue: T): T;
}
```

### ToolDefinition

Abstracts `vscode.LanguageModelChatTool`.

```tsx
export interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
}

export interface ToolCallResult {
  toolName: string;
  input: Record<string, unknown>;
  result: string;
}
```

## 3. Module Refactoring Plan

Each affected module is modified to accept a provider via constructor or dependency injection.

### model-resolver.ts

**Before:**

```tsx
import * as vscode from 'vscode';

export class ModelResolver {
  async selectModel(tier: ModelTier): Promise<string> {
    const models = await vscode.lm.selectChatModels({ /* ... */ });
    // ...
  }
}
```

**After:**

```tsx
import { ModelProvider, ModelSelector, ModelInfo } from '../providers';

export class ModelResolver {
  constructor(private modelProvider: ModelProvider) {}

  async selectModel(tier: ModelTier): Promise<string> {
    const models = await this.modelProvider.selectModels({ /* ... */ });
    // ... same logic, different data source
  }
}
```

### agent-spawner.ts

**Before:** Calls `vscode.lm.sendChatRequest()` directly.

**After:** Calls `this.modelProvider.sendRequest()`. Same three-layer prompt construction, same tool scoping, same result aggregation.

### workflow-engine.ts

**Before:** Accepts `vscode.ChatResponseStream` and `vscode.CancellationToken` in `WorkflowContext`.

**After:** Accepts `ProgressReporter` and `CancellationHandle` in `WorkflowContext`.

**Updated WorkflowContext:**

```tsx
interface WorkflowContext {
  prompt: string;
  intent: ClassificationResult;
  projectModel: ProjectModel;
  progress: ProgressReporter;       // was: chatResponseStream
  cancellation: CancellationHandle;  // was: cancellationToken
  previousStepResults?: StepResult[];
}
```

> **Note:** This changes the `WorkflowContext` type in `types.ts`. The Chat Participant creates a `WorkflowContext` using `VSCodeProgressReporter` and `VSCodeCancellationHandle` wrappers. The MCP server creates one using `StderrProgressReporter` and `NullCancellationHandle`.
> 

### file-generator.ts

**Before:** Calls `vscode.workspace.textDocuments.some(...)` to check if file is open.

**After:** Calls `this.fileSystem.isFileOpenInEditor(filePath)`. In standalone mode, always returns `false` (no open editor → no deferred writes needed).

## 4. Provider Implementations

### VS Code Providers (`src/shell/vscode-providers.ts`)

Wraps real VS Code APIs. Used when `RuntimeMode === 'extension'`.

| Provider | VS Code API Wrapped |
| --- | --- |
| `VSCodeModelProvider` | `vscode.lm.selectChatModels()`, `model.sendRequest()` |
| `VSCodeProgressReporter` | `vscode.ChatResponseStream` |
| `VSCodeCancellationHandle` | `vscode.CancellationToken` |
| `VSCodeFileSystemProvider` | `vscode.workspace.fs`, `vscode.workspace.textDocuments` |
| `VSCodeConfigProvider` | `vscode.workspace.getConfiguration('roadie')` |

### Standalone Providers (`src/mcp/standalone-providers.ts`)

Node.js implementations. Used when `RuntimeMode === 'standalone'`.

| Provider | Implementation |
| --- | --- |
| `NullModelProvider` | Throws on `sendRequest()` — client provides LLM |
| `DirectAPIModelProvider` | Direct HTTP to Anthropic/OpenAI API (opt-in) |
| `StderrProgressReporter` | Writes to `process.stderr` |
| `NullCancellationHandle` | `isCancelled` always `false` |
| `NodeFileSystemProvider` | `fs.promises` for all file operations |
| `FileConfigProvider` | Reads `.vscode/settings.json`  • env vars |

## 5. Container Updates

`container.ts` is updated to accept `RuntimeMode` and wire the appropriate providers.

```tsx
export type RuntimeMode = 'extension' | 'standalone';

export interface ContainerConfig {
  projectRoot: string;
  dbPath: string;
  apiKey?: string;
  apiProvider?: 'anthropic' | 'openai';
}

export function createContainer(
  mode: RuntimeMode,
  config: ContainerConfig
): Container {
  // Wire providers based on mode
  // Wire core modules (same for both modes)
  // Return container
}
```

## 6. Refactoring Effort Summary

| Module | Change Description | Effort | Risk |
| --- | --- | --- | --- |
| `providers.ts` | New file — interface definitions | S | None |
| `shell/vscode-providers.ts` | New file — VS Code implementations | S | None |
| `mcp/standalone-providers.ts` | New file — standalone implementations | S | None |
| `engine/model-resolver.ts` | Accept `ModelProvider` in constructor | S | Low |
| `spawner/agent-spawner.ts` | Accept `ModelProvider` in constructor | S | Low |
| `engine/workflow-engine.ts` | Accept `ProgressReporter`  • `CancellationHandle` | S | Low |
| `engine/step-executor.ts` | Accept `CancellationHandle` | S | Low |
| `generator/file-generator.ts` | Accept `FileSystemProvider` | S | Low |
| `spawner/tool-registry.ts` | Accept `ToolDefinition[]` | S | Low |
| `container.ts` | Add `RuntimeMode`  • provider wiring | M | Medium |
| `types.ts` | Update `WorkflowContext` type | S | Low |

**Total effort:** ~4–6 hours. The key risk is ensuring all Phase 1/1.5 tests pass after refactoring (Step 9 gate).

## 7. Validation Criteria

1. All existing Phase 1/1.5 unit tests pass without modification (providers are injected via DI, tests use mocks)
2. All existing integration tests pass
3. `container.ts` with `mode='extension'` produces identical behavior to pre-refactoring
4. `container.ts` with `mode='standalone'` creates a functional container without importing `vscode`
5. `NullModelProvider.sendRequest()` throws with informative error message
6. `NodeFileSystemProvider` reads/writes files correctly
7. `FileConfigProvider` reads `roadie.*` settings from `.vscode/settings.json`
8. `FileConfigProvider` falls back to env vars when settings file doesn't exist