# 🖥️ Standalone Mode Design

## Running Roadie Without VS Code

---

## 1. CLI Entry Point

**File:** `bin/roadie-mcp.ts`

**Usage:**

```bash
npx roadie-mcp --project /path/to/project
npx roadie-mcp --project . --api-key sk-ant-...
npx roadie-mcp -p . 2>/dev/null  # suppress logs for clean stdio
```

**Arguments:**

| Argument | Short | Default | Description |
| --- | --- | --- | --- |
| `--project` | `-p` | `.` | Project root path |
| `--db` | `-d` | `{project}/.github/.roadie/project-model.db` | Database file path |
| `--api-key` |  | `$ANTHROPIC_API_KEY` or `$OPENAI_API_KEY` | API key for standalone LLM |
| `--api-provider` |  | `anthropic` | Which LLM provider (`anthropic` or `openai`) |

**Implementation:**

```tsx
#!/usr/bin/env node
import { parseArgs } from 'node:util';
import * as path from 'path';
import { RoadieMCPServer } from '../src/mcp/server';

const { values } = parseArgs({
  options: {
    project: { type: 'string', short: 'p', default: '.' },
    db: { type: 'string', short: 'd' },
    'api-key': { type: 'string' },
    'api-provider': { type: 'string', default: 'anthropic' },
  },
  strict: true,
});

const projectRoot = path.resolve(values.project ?? '.');
const server = new RoadieMCPServer({
  projectRoot,
  dbPath: values.db,
  mode: 'standalone',
  apiKey: values['api-key'] ?? process.env.ANTHROPIC_API_KEY ?? process.env.OPENAI_API_KEY,
  apiProvider: values['api-provider'] as 'anthropic' | 'openai',
});

process.on('SIGINT', async () => { await server.stop(); process.exit(0); });
process.on('SIGTERM', async () => { await server.stop(); process.exit(0); });

server.start().catch((err) => {
  process.stderr.write(`Failed to start Roadie MCP server: ${err.message}\n`);
  process.exit(1);
});
```

## 2. Startup Sequence

1. Parse CLI arguments
2. Resolve project root to absolute path
3. Resolve database path: `{projectRoot}/.github/.roadie/project-model.db`
4. If database directory doesn't exist, create `.github/.roadie/`
5. Open SQLite with `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000`
6. Run schema migrations if needed (`schema_version` table)
7. Load project model from SQLite into memory
8. If model is empty (no `tech_stack` entries) or stale (`lastAnalyzed` > 24h):
    - Run `projectAnalyzer.analyze('full')` automatically
    - Log to stderr: `[roadie] Project model empty/stale, running initial analysis...`
9. Create `RoadieMCPServer` with `mode: 'standalone'`
10. Register all 10 MCP tool handlers
11. Connect `StdioServerTransport` and begin accepting requests
12. Log to stderr: `[roadie] MCP server ready. 10 tools available.`

**What does NOT happen:**

- No file watcher started (no VS Code `FileSystemWatcher` available)
- No Chat Participant registered (no VS Code chat API)
- No sidebar or status bar (no VS Code UI)
- No edit tracking from file watcher (only on explicit `generate_file` calls)

## 3. Feature Degradation Matrix

| Feature | Extension Mode | Standalone Mode | Degradation Strategy |
| --- | --- | --- | --- |
| Project analysis | ✅ Full | ✅ Full | Identical |
| Project context | ✅ Full | ✅ Full | Identical |
| File generation | ✅ Full | ✅ Full | No deferred writes (always writes immediately) |
| Pattern queries | ✅ Full | ✅ Full | Identical |
| Workflow history | ✅ Full | ✅ Full | Identical |
| Recommendations | ✅ Full | ✅ Full | Identical |
| Project rescan | ✅ Full | ✅ Full | Identical |
| Workflow execution | ✅ Full | ⚠️ Conditional | Requires API key env var |
| Feature plan approval | ✅ `stream.button()` | ⚠️ Auto-approve | No interactive UI |
| File watching | ✅ VS Code watcher | ❌ Not available | Use `rescan_project` manually |
| Chat Participant | ✅ @roadie | ❌ Not available | MCP client provides chat |
| Sidebar/status bar | ✅ UI | ❌ Not available | VS Code only |
| Edit tracking | ✅ On file change | ⚠️ On generate only | No watcher to detect external edits |
| Deferred writes | ✅ Checks open editors | ❌ Always writes | No editor to check |
| Config source | `.vscode/settings.json` | Settings + env vars | `ROADIE_*` env vars override |

## 4. LLM Access Strategy

### Default: Client Provides LLM (Option C)

The MCP client (Claude Code, Gemini CLI, etc.) already has its own model. Roadie provides tools and project intelligence; the client provides reasoning.

**In practice:**

- 9 of 10 tools work perfectly without any LLM
- Only `roadie/run_workflow` needs an LLM
- When called without a model provider, returns:
    
    ```json
    {
      "error": "Workflow execution requires an LLM. Configure ANTHROPIC_API_KEY or OPENAI_API_KEY, or use an MCP client that provides its own model.",
      "code": "LLM_UNAVAILABLE"
    }
    ```
    

### Opt-In: Direct API Keys (Option B)

For CI/CD pipelines or scenarios where autonomous workflow execution is needed:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
npx roadie-mcp --project .
```

This enables the `DirectAPIModelProvider`, which makes direct HTTP calls to the LLM provider. The model resolver maps tiers to available models:

| Tier | Anthropic Model | OpenAI Model |
| --- | --- | --- |
| free | claude-3-5-haiku | gpt-4o-mini |
| standard | claude-sonnet-4 | gpt-4o |
| premium | claude-opus-4 | gpt-4o (same) |

> **Note:** `DirectAPIModelProvider` is interface-only in Phase 2. The actual HTTP implementation is deferred to a follow-up. Phase 2 defines the interface and throws "not yet implemented" if used. This is acceptable because the primary standalone use case (Claude Code providing tools) doesn't need it.
> 

## 5. Environment Variables

| Variable | Purpose | Default |
| --- | --- | --- |
| `ROADIE_PROJECT_ROOT` | Project root path | cwd |
| `ROADIE_DB_PATH` | Database file path | Auto-detected |
| `ANTHROPIC_API_KEY` | Anthropic API key | Not set |
| `OPENAI_API_KEY` | OpenAI API key | Not set |
| `ROADIE_TESTCOMMAND` | Override test command | Auto-detected |
| `ROADIE_TESTTIMEOUT` | Test timeout (seconds) | 300 |
| `ROADIE_EDITTRACKING` | Enable edit tracking | false |
| `ROADIE_WORKFLOWHISTORY` | Enable workflow history | false |
| `ROADIE_AUTOCOMMIT` | Auto-commit generated files | false |
| `ROADIE_MODELPREFERENCE` | Model preference | null (balanced) |

All `roadie.*` settings from `.vscode/settings.json` can be set via `ROADIE_*` env vars. Env vars take precedence over file settings.

## 6. Graceful Shutdown

1. `SIGINT` or `SIGTERM` received
2. MCP server stops accepting new requests
3. Active tool calls are allowed to complete (up to 10s grace period)
4. Database connections are closed
5. Process exits with code 0

If the process is killed ungracefully (`SIGKILL`), SQLite WAL mode ensures database integrity.