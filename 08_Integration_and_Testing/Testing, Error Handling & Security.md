# 🧹 Testing, Error Handling & Security

## Testing Strategy, Error Handling, Performance, and Security

---

## 1. Testing Strategy

### Unit Tests

| Module | What Is Tested | Mock Strategy |
| --- | --- | --- |
| `providers.ts` | Interface contract validation (compile-time) | N/A |
| `standalone-providers.ts` | `NullModelProvider` throws on `sendRequest`, `FileConfigProvider` reads settings, `NodeFileSystemProvider` R/W | Fixture files |
| `mcp/server.ts` | Tool registration, request routing, error formatting | Mock container |
| `mcp/tools/project-tools.ts` | Input validation, output formatting, analyzer integration | Mock ProjectAnalyzer, ProjectModel |
| `mcp/tools/workflow-tools.ts` | Workflow triggering, status, standalone degradation | Mock WorkflowEngine |
| `mcp/tools/generator-tools.ts` | File generation, force flag, status reporting | Mock FileGenerator |
| `mcp/tools/query-tools.ts` | Pattern filtering, history retrieval, recommendations | Mock ProjectModel, LearningDB |
| `shell/mcp-manager.ts` | Process spawn, crash restart, graceful shutdown | Mock child_process |
| `shell/vscode-providers.ts` | VS Code API wrapping | Mock vscode APIs |

### Integration Tests

1. **Standalone startup:** `npx roadie-mcp --project test/fixtures/node-js-next-js` → server starts, responds to tools, exits cleanly
2. **Extension-spawned:** Extension activates → MCP manager spawns server → server responds → deactivation → clean exit
3. **Concurrent SQLite access:** Extension writes while MCP server reads → both succeed, no corruption
4. **Provider refactoring regression:** ALL Phase 1/1.5 tests pass unchanged
5. **Cross-tool config:** `generate_all_files` → `.mcp.json` created → valid JSON → correct server entry
6. **MCP.json merge:** Existing `.mcp.json` with other servers + `generate_file('mcp-config')` → Roadie added, others preserved

### Test File Co-location

Following Phase 1 pattern:

```
src/mcp/server.ts
src/mcp/server.test.ts

src/mcp/tools/project-tools.ts
src/mcp/tools/project-tools.test.ts

[etc.]
```

---

## 2. Error Handling

### Error Response Format

All MCP tools return errors as JSON in the MCP `content` array with `isError: true`:

```tsx
// In server.ts CallToolRequestSchema handler
try {
  const result = await this.executeTool(name, args);
  return { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] };
} catch (error) {
  const message = error instanceof Error ? error.message : String(error);
  return {
    content: [{ type: 'text', text: JSON.stringify({ error: message, code: getErrorCode(error) }) }],
    isError: true
  };
}
```

### Error Code Registry

| Code | HTTP Analog | When |
| --- | --- | --- |
| `PROJECT_NOT_FOUND` | 404 | Project root doesn't exist |
| `MODEL_EMPTY` | 422 | Project model not yet built |
| `MODEL_STALE` | 422 | Model older than 24 hours (warning, not blocking) |
| `LLM_UNAVAILABLE` | 503 | No model provider in standalone mode |
| `WORKFLOW_NOT_FOUND` | 404 | Unknown workflow type |
| `WORKFLOW_FAILED` | 500 | Workflow failed after retries |
| `EXECUTION_NOT_FOUND` | 404 | Unknown execution ID |
| `FILE_PERMISSION_ERROR` | 403 | Cannot write to file system |
| `DATABASE_ERROR` | 500 | SQLite error |
| `INVALID_INPUT` | 400 | Zod validation failed |
| `HISTORY_DISABLED` | 422 | workflowHistory not enabled |
| `ANALYSIS_TIMEOUT` | 504 | Analysis exceeded 30s |

### Input Validation

Every tool validates inputs with Zod before execution:

```tsx
const AnalyzeProjectInput = z.object({
  scope: z.enum(['full', 'dependencies', 'patterns', 'structure']).default('full'),
  force: z.boolean().default(false),
});

export async function handleAnalyzeProject(
  container: Container,
  args: Record<string, unknown>
): Promise<unknown> {
  const input = AnalyzeProjectInput.parse(args); // throws ZodError on invalid
  // ...
}
```

Zod errors are caught at the server level and returned with `INVALID_INPUT` code.

---

## 3. Performance Budgets

| Operation | Budget | Enforcement |
| --- | --- | --- |
| MCP server startup (standalone) | <3s | Measured in integration test |
| `analyze_project` (full) | <30s | Same as Phase 1 |
| `get_project_context` | <100ms | In-memory serialization |
| `generate_file` (single) | <1s | Template + file write |
| `generate_all_files` | <2s | Parallel generators |
| `query_patterns` | <50ms | SQLite query |
| `query_workflow_history` | <50ms | SQLite query |
| `get_recommendations` | <200ms | Model inspection + file checks |
| `rescan_project` | <30s | Full re-analysis |
| MCP JSON-RPC overhead | <10ms | Serialization + transport |
| Tool input validation | <1ms | Zod parse |

---

## 4. Security Considerations

### Standalone Mode Security

- MCP server runs with user permissions. No privilege escalation.
- API keys passed via environment variables, never written to disk by Roadie.
- `.mcp.json` contains commands only, never API keys.
- SQLite database contains no secrets (no code content, no credentials).

### Tool Safety Classification

| Tool | Safety Level | Reason |
| --- | --- | --- |
| `get_project_context` | Safe | Read-only, no file access |
| `query_patterns` | Safe | Read-only, no file access |
| `query_workflow_history` | Safe | Read-only, no file access |
| `get_recommendations` | Safe | Read-only, checks file existence |
| `get_workflow_status` | Safe | Read-only, in-memory lookup |
| `analyze_project` | Low risk | Reads files, writes to internal DB only |
| `rescan_project` | Low risk | Same as analyze |
| `generate_file` | Medium risk | Writes to `.github/` directory |
| `generate_all_files` | Medium risk | Writes multiple files |
| `run_workflow` | High risk | Modifies source files, runs shell commands |

The MCP client is responsible for confirming with the user before calling medium/high risk tools.

### Sensitive Content Exclusion

Same rules as Phase 1 (TAD §9.3):

- `.env` values never in tool outputs or `toContext()`
- Private keys, API keys, connection strings excluded
- Directories named `secrets/`, `credentials/`, `private/` excluded

### Workspace Trust

In standalone mode, there is no VS Code workspace trust check. The MCP client is responsible for user consent. `run_workflow` can execute shell commands (test runner), so the MCP client should confirm before invoking.

---

## 5. Configuration (No New Settings)

Phase 2 adds no new VS Code settings. The MCP server reads existing `roadie.*` settings from `.vscode/settings.json` in both modes.

In standalone mode, settings can also be set via environment variables using the `ROADIE_*` prefix (see Standalone Mode Design page).

---

## 6. Dependencies

### New npm Dependency

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  }
}
```

### Existing Dependencies (unchanged)

- `better-sqlite3` — SQLite access
- `zod` — Schema validation
- `fast-glob` — Directory scanning

### No New Dev Dependencies

Existing test infrastructure (Vitest, mock layer) is sufficient for Phase 2 testing.