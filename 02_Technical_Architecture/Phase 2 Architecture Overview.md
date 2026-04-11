# 🏗️ Phase 2 Architecture Overview

## Big Picture: How Phase 2 Components Work Together

---

## 1. Component Diagram

```
┌───────────────────────────────────────────────────────────────┐
│  VS Code Extension Shell                                         │
│  (chat-participant, sidebar, status-bar, file-watcher)          │
│                                                                   │
│    ┌──────────────────────┐     ┌──────────────────────┐        │
│    │ MCPManager           │────▶│ MCP Server Process   │        │
│    │ (spawns + manages)   │     │ (stdio transport)    │        │
│    └──────────────────────┘     └──────────┬───────────┘        │
└─────────────────────────────────────────────┼───────────────────┘
                                              │
          ┌───────────────────────────────────┼────────────────┐
          │  Core Engine (VS Code–independent)                  │
          │                                                      │
          │  Project Analyzer  │ Project Model  │ File Generator  │
          │  Workflow Engine   │ Agent Spawner  │ Learning DB     │
          │  Intent Classifier │ Section Mgr    │                 │
          └─────────────────────────────────────────────────────┘

External MCP Clients (standalone mode):
  Claude Code  │  Gemini CLI  │  Cursor  │  CI/CD Pipeline
       │              │            │            │
       └──────────────┴──────────────┴────────────┘
                       │ stdio
              ┌────────▼────────┐
              │ MCP Server      │
              │ (standalone)    │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ Core Engine     │
              └─────────────────┘
```

## 2. Process Model

### Extension-Spawned Mode

The VS Code extension spawns the MCP server as a child process via `child_process.spawn()`. Communication uses stdio (stdin/stdout JSON-RPC). The extension manages the server lifecycle:

1. **Start:** Extension activation → `MCPManager.start()` → `child_process.spawn('node', [serverScript])`
2. **Communication:** JSON-RPC over stdin/stdout. stderr used for logging.
3. **Crash recovery:** If server process exits unexpectedly, MCPManager restarts with exponential backoff (1s, 2s, 3s). Max 3 restart attempts.
4. **Stop:** Extension deactivation → `MCPManager.stop()` → `SIGTERM` to child process.
5. **State sharing:** Both processes access the same SQLite database file.

### Standalone Mode

A developer or MCP client runs `npx roadie-mcp --project .` from the command line:

1. Parse CLI arguments (project root, optional db path, optional API key).
2. Resolve database path: `{projectRoot}/.github/.roadie/project-model.db`.
3. If database does not exist, create `.github/.roadie/` directory and initialize empty database.
4. Open database with `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000`.
5. Load project model from SQLite into memory.
6. If model is empty or stale (no `tech_stack` entries, or `lastAnalyzed` > 24h), run `projectAnalyzer.analyze('full')`.
7. Register MCP tool handlers.
8. Connect stdio transport and begin accepting requests.
9. Do NOT start file watcher (no VS Code FileSystemWatcher).
10. Do NOT register Chat Participant (no VS Code chat API).

## 3. Data Flow

### Read-Only Tool Call (e.g., `get_project_context`)

```
MCP Client ── stdio ──▶ MCP Server ──▶ ProjectModel.toContext()
                                         │
                                    SQLite Read
                                         │
                                    ◀── JSON response
```

### Mutating Tool Call (e.g., `generate_file`)

```
MCP Client ── stdio ──▶ MCP Server ──▶ FileGenerator.generate()
                                         │
                                    Read project model
                                    Generate content
                                    Check section ownership
                                    Hash comparison
                                    Write file (if changed)
                                    Record snapshot in SQLite
                                         │
                                    ◀── JSON response (status, path, hash)
```

### Workflow Tool Call (e.g., `run_workflow`)

```
MCP Client ── stdio ──▶ MCP Server ──▶ WorkflowEngine.execute()
                                         │
                                    Requires ModelProvider!
                                    NullModelProvider → ERROR
                                    DirectAPIModelProvider → API calls
                                    VSCodeModelProvider → vscode.lm
                                         │
                                    Steps execute sequentially
                                    Each step → AgentSpawner → LLM call
                                    Results accumulated
                                         │
                                    ◀── JSON response (full workflow result)
```

## 4. SQLite Concurrent Access

Both the extension and MCP server may access `project-model.db` simultaneously.

**Strategy:**

- SQLite opened in **WAL mode** by both processes (allows concurrent readers + single writer)
- `PRAGMA busy_timeout=5000` (5 second retry on lock contention)
- `better-sqlite3` uses SQLite's built-in file-level locking
- Extension writes every 5 seconds (debounced). MCP server writes only on explicit mutations.
- No cross-process signaling. Each process reads fresh on each operation.
- In-memory cache staleness bounded by 5-second debounce interval — acceptable.

**Write contention analysis:**

| Writer | Frequency | Duration | Risk |
| --- | --- | --- | --- |
| Extension (model flush) | Every 5s | <50ms | Low |
| Extension (snapshot) | Per generation | <20ms | Low |
| MCP (analyze) | On-demand | <500ms | Low |
| MCP (generate) | On-demand | <100ms | Low |
| MCP (workflow log) | Per workflow | <20ms | Low |

Maximum realistic contention: two writes within 5-second busy_timeout window. This is well within SQLite's capability.

## 5. Module Dependency Graph (Phase 2)

```
extension.ts
    │
    ├── shell/mcp-manager.ts ──▶ spawns MCP server process
    ├── shell/vscode-providers.ts ──▶ implements providers for extension
    │
    └── container.ts (mode='extension')
            │
            ├── providers.ts (interfaces)
            ├── model/database.ts
            ├── model/project-model.ts
            ├── analyzer/project-analyzer.ts
            ├── engine/model-resolver.ts ──▶ accepts ModelProvider
            ├── spawner/agent-spawner.ts ──▶ accepts ModelProvider
            ├── engine/workflow-engine.ts ──▶ accepts ProgressReporter
            ├── generator/file-generator.ts ──▶ accepts FileSystemProvider
            └── learning/learning-database.ts

bin/roadie-mcp.ts
    │
    └── mcp/server.ts
            │
            ├── mcp/standalone-providers.ts ──▶ implements providers for standalone
            ├── container.ts (mode='standalone')
            │       │
            │       └── (same core modules as extension)
            │
            ├── mcp/tools/project-tools.ts
            ├── mcp/tools/workflow-tools.ts
            ├── mcp/tools/generator-tools.ts
            └── mcp/tools/query-tools.ts
```

Dependencies flow in one direction: MCP tools → core modules → database. No circular dependencies.