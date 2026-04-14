# 📊 Build Order & Dependencies

## 20-Step Build Sequence with Verification Gates

---

## Build Order

| Step | Module | What to Build | Depends On | Effort | Milestone |
| --- | --- | --- | --- | --- | --- |
| 1 | `src/providers.ts` | Provider interface definitions | types.ts | S | M21 |
| 2 | `src/shell/vscode-providers.ts` | VS Code provider implementations | providers.ts | S | M21 |
| 3 | `src/mcp/standalone-providers.ts` | Standalone provider implementations | providers.ts | S | M21 |
| 4 | `src/engine/model-resolver.ts` | Refactor: accept `ModelProvider` | providers.ts | S | M21 |
| 5 | `src/spawner/agent-spawner.ts` | Refactor: accept `ModelProvider` | providers.ts | S | M21 |
| 6 | `src/engine/workflow-engine.ts` | Refactor: accept `ProgressReporter`  • `CancellationHandle` | providers.ts | S | M21 |
| 7 | `src/engine/step-executor.ts` | Refactor: accept `CancellationHandle` | providers.ts | S | M21 |
| 8 | `src/generator/file-generator.ts` | Refactor: accept `FileSystemProvider` | providers.ts | S | M21 |
| **9** | **ALL TESTS** | **🚨 GATE: Run all Phase 1/1.5 tests — MUST PASS** | Steps 1–8 | — | M21 |
| 10 | `src/container.ts` | Update: `RuntimeMode`  • provider wiring | providers.ts, all refactored | M | M21 |
| 11 | `src/mcp/server.ts` | MCP server scaffold (initialize + ping) | container.ts | S | M21 |
| 12 | `bin/roadie-mcp.ts` | CLI entry point | server.ts | S | M21 |
| 13 | `src/shell/mcp-manager.ts` | Extension-side process manager | vscode | S | M21 |
| 14 | `src/mcp/tools/project-tools.ts` | `analyze_project`, `get_project_context`, `rescan_project` | server, analyzer, model | S | M22 |
| 15 | `src/mcp/tools/query-tools.ts` | `query_patterns`, `query_workflow_history`, `get_recommendations` | server, model, learning | S | M22 |
| 16 | `src/mcp/tools/generator-tools.ts` | `generate_file`, `generate_all_files` | server, file-generator | S | M23 |
| 17 | `src/mcp/tools/workflow-tools.ts` | `run_workflow`, `get_workflow_status` | server, workflow-engine | M | M23 |
| 18 | `src/generator/templates/mcp-config.ts` | `.mcp.json` generator | file-generator | S | M23 |
| 19 | `src/generator/templates/agent-definitions.ts` | Update: MCP integration section in [AGENTS.md](http://AGENTS.md) | existing template | S | M23 |
| 20 | Integration testing + `tsup.config.ts` update | End-to-end verification | all | M | M23 |
| **21** | `src/generator/templates/claude-hooks.ts` | Claude Code Hooks generator — `.claude/settings.json` with SessionStart/PostToolUse/Stop hooks; new `prime`, `observe`, `reconcile` CLI subcommands; update `generate_all_files` to call hooks generator | Steps 16, 18 | S | M23 |

---

## Step 9: The Gate

**This is the most important step.** After refactoring 5 modules to accept providers (Steps 4–8), ALL existing Phase 1 and Phase 1.5 tests must pass.

**Why:** Provider refactoring changes internal implementation. If any test breaks, the refactoring introduced a behavioral change. Fix the refactoring, not the tests.

**How to verify:**

```bash
npm run test        # All Vitest tests
npm run lint        # ESLint
npm run build       # TypeScript compilation
```

All three must exit 0. If any fail, do not proceed to Step 10.

---

## Build Prompts (for AI Agents)

### Step 1: providers.ts

> Create `src/providers.ts` with the following TypeScript interfaces: `ModelProvider` (selectModels, sendRequest), `ModelSelector`, `ModelInfo`, `ChatMessage`, `ModelRequestOptions`, `ModelResponse`, `ToolDefinition`, `ToolCallResult`, `ProgressReporter` (report, reportMarkdown), `CancellationHandle` (isCancelled, onCancelled), `FileSystemProvider` (isFileOpenInEditor, readFile, writeFile, fileExists), `ConfigProvider` (get). Add JSDoc module header. No implementation — interfaces only. See Core/Shell Split page for exact interface definitions.
> 

### Step 4: model-resolver.ts refactor

> Refactor `src/engine/model-resolver.ts` to accept a `ModelProvider` in its constructor instead of calling `vscode.lm.selectChatModels()` directly. Replace all `vscode.lm.selectChatModels()` calls with `this.modelProvider.selectModels()`. Map the returned `ModelInfo[]` to the existing tier selection logic. Do NOT change any public interface. Update the corresponding test file to inject a mock `ModelProvider`.
> 

### Step 5: agent-spawner.ts refactor

> Refactor `src/spawner/agent-spawner.ts` to accept a `ModelProvider` in its constructor instead of calling `vscode.lm.sendChatRequest()` directly. Replace all LLM call sites with `this.modelProvider.sendRequest()`. The three-layer prompt construction, tool scoping, and result aggregation logic remain identical. Update tests to inject a mock `ModelProvider`.
> 

### Step 11: MCP server scaffold

> Create `src/mcp/server.ts` using `@modelcontextprotocol/sdk`. The server uses `StdioServerTransport`. It accepts an `MCPServerConfig` (projectRoot, dbPath, mode, apiKey, apiProvider). On construction, it creates a container via `createContainer(mode, config)`. It registers `ListToolsRequestSchema` and `CallToolRequestSchema` handlers. The `CallToolRequestSchema` handler routes tool calls by name to handler functions. Expose `start()` and `stop()` methods. Add a `server.test.ts` with mock container.
> 

### Step 17: workflow-tools.ts

> Create `src/mcp/tools/workflow-tools.ts` with two handler functions: `handleRunWorkflow` and `handleGetWorkflowStatus`. `handleRunWorkflow` validates input with Zod, creates a `WorkflowContext` with `StderrProgressReporter` and `NullCancellationHandle`, calls `workflowEngine.execute()`, and returns the result as JSON. If ModelProvider is NullModelProvider, return an error with code `LLM_UNAVAILABLE`. The feature workflow's `autoApprove` option defaults to `true` in standalone mode. `handleGetWorkflowStatus` looks up active workflows by execution ID.
> 

### Step 21: claude-hooks.ts (Claude Code Hooks Generator)

> Create `src/generator/templates/claude-hooks.ts`. Export `generateClaudeHooks(model: ProjectModel): string` that returns the JSON content for `.claude/settings.json` containing all three lifecycle hooks: `SessionStart` (runs `npx roadie-mcp prime --project .`), `PostToolUse` with matcher `Edit|Write|MultiEdit` (runs `npx roadie-mcp observe --tool $TOOL --file $FILE`), and `Stop` (runs `npx roadie-mcp reconcile --project .`). Add a `mergeOrCreateClaudeHooks(existingPath, roadieHooks, fs)` helper that reads existing JSON, performs append-only merge of the hooks arrays (never overwrites existing entries, deduplicates by command string), and returns the merged JSON string. On invalid JSON, throw with a clear message. Register `'claude-hooks'` as a file type in `file-generator.ts` with output path `.claude/settings.json`. Update `generate_all_files` to include this type in its generation pass.
>
> Add three CLI subcommands to `bin/roadie-mcp.ts`: `prime --project <path>` (loads ProjectModel into memory, warms caches, exits 0), `observe --tool <name> --file <path>` (writes edit event directly to SQLite `learning_events` table using `NodeFileSystemProvider`, exits 0), `reconcile --project <path>` (calls `LearningEngine.reconcile()`, commits WAL, exits 0). All three subcommands must write only to stderr, never stdout, and must always exit with code 0 — hook failures must not interrupt Claude Code sessions.

---

## Milestone Boundaries

### M21: MCP Server Scaffold (Steps 1–13)

**Deliverables:** Provider interfaces, all module refactoring, MCP server responding to `initialize` and `ping`, CLI entry point, extension-side process manager.

**Validation:**

1. All Phase 1/1.5 tests pass (Step 9 gate)
2. `node out/bin/roadie-mcp.js --project .` starts and responds to `initialize`
3. Extension spawns MCP server on activation
4. Extension restarts MCP server on crash (up to 3 times)
5. `npm run build` produces both bundles

### M22: Project Tools (Steps 14–15)

**Deliverables:** 6 read-only tools (`analyze_project`, `get_project_context`, `rescan_project`, `query_patterns`, `query_workflow_history`, `get_recommendations`).

**Validation:**

1. Each tool returns correct JSON on valid input
2. Each tool returns error JSON on invalid input
3. All tools respond within 2s on typical Node.js project
4. Tools work identically in extension-spawned and standalone modes

### M23: Workflow + Generator Tools (Steps 16–21)

**Deliverables:** 4 mutating tools (`generate_file`, `generate_all_files`, `run_workflow`, `get_workflow_status`), `.mcp.json` generator, [AGENTS.md](http://AGENTS.md) MCP section, Claude Code Hooks generator (`claude-hooks.ts`), `prime`/`observe`/`reconcile` CLI subcommands, integration tests.

**Validation:**

1. `generate_file` creates files with section ownership markers
2. `generate_all_files` generates all file types (including `.claude/settings.json`)
3. `run_workflow` returns error when no LLM available
4. `run_workflow` executes workflow with mock LLM in tests
5. `.mcp.json` is generated with correct server config
6. `.mcp.json` merges with existing entries
7. Claude Code can discover and connect via generated config
8. End-to-end: standalone server → tool call → result
9. `.claude/settings.json` generated with all 3 hooks (SessionStart, PostToolUse, Stop)
10. `.claude/settings.json` append-only merge: existing non-Roadie hook entries are preserved
11. `prime`/`observe`/`reconcile` subcommands exit 0 and produce no stdout output

---

## Estimated Time per Step

| Steps | Description | Hours |
| --- | --- | --- |
| 1–3 | Provider interfaces + implementations | 2–3 |
| 4–8 | Module refactoring | 3–4 |
| 9 | Test gate | 0.5 |
| 10–13 | Container + MCP scaffold + CLI + manager | 3–4 |
| 14–15 | Read-only MCP tools | 2–3 |
| 16–17 | Mutating MCP tools | 2–3 |
| 18–19 | Cross-tool config generators | 1–2 |
| 20 | Integration testing + build config | 2–3 |
| 21 | Claude Code Hooks generator + CLI subcommands | 1–2 |
| **Total** |  | **16–22** |