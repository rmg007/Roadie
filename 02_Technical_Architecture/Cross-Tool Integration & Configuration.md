# 🔗 Cross-Tool Integration & Configuration

## Claude Code, Gemini CLI, Cursor, and .mcp.json Configuration

---

## 1. Claude Code Integration

### Configuration

Claude Code discovers MCP servers via `.mcp.json` at the project root or `~/.claude/settings.json` globally.

**Project-level `.mcp.json` (recommended):**

```json
{
  "mcpServers": {
    "roadie": {
      "command": "npx",
      "args": ["roadie-mcp", "--project", "."],
      "env": {}
    }
  }
}
```

**Global `~/.claude/settings.json`:**

```json
{
  "mcpServers": {
    "roadie": {
      "command": "npx",
      "args": ["roadie-mcp", "--project", "."],
      "env": {}
    }
  }
}
```

### Developer Experience

1. Developer has Roadie installed (VS Code extension or `npm install -g roadie-mcp`)
2. Developer runs `claude` in project directory
3. Claude Code reads `.mcp.json`, spawns `npx roadie-mcp --project .`
4. Roadie loads project model, exposes 10 tools
5. Claude Code can now:
    - Call `roadie/get_project_context` to understand the project before answering questions
    - Call `roadie/analyze_project` to scan structure
    - Call `roadie/generate_all_files` to create/update AI config files
    - Call `roadie/query_patterns` to learn coding conventions
    - Call `roadie/get_recommendations` for improvement suggestions
6. Claude provides the LLM intelligence; Roadie provides project-aware tools

### Example Claude Code Session

```
$ claude
> What's the tech stack of this project?

[Claude calls roadie/analyze_project]

This is a Next.js 14 project using TypeScript, Prisma ORM with PostgreSQL,
and Vitest for testing. It uses pnpm as the package manager...

> Generate AI configuration files for this project

[Claude calls roadie/generate_all_files]

I've generated 8 configuration files:
- .github/copilot-instructions.md (created)
- AGENTS.md (created)
- .github/instructions/typescript.instructions.md (created)
...
```

## 2. Claude Code Hooks Integration (Autonomous Mode)

The `.mcp.json` approach (Section 1) exposes Roadie's tools passively — Claude Code discovers the server and a developer can call tools manually. Hooks take this further: Roadie generates a `.claude/settings.json` that registers **lifecycle hooks**, so Claude Code fires Roadie automatically at key moments without any LLM instruction.

**Philosophy:** install Roadie once → everything else is invisible.

### The Three Hooks

| Hook Event | CLI Subcommand | When It Fires | What It Does |
| --- | --- | --- | --- |
| `SessionStart` | `roadie-mcp prime --project .` | Before the first user turn | Warms SQLite-backed context so the first tool call is fast |
| `PostToolUse` (Edit\|Write\|MultiEdit) | `roadie-mcp observe --tool $TOOL --file $FILE` | After every file-editing tool | Tracks edits directly to SQLite without going through MCP protocol |
| `Stop` | `roadie-mcp reconcile --project .` | After the session ends | Runs end-of-session learning reconciliation against the project model |

All three subcommands are **fire-and-forget**: no stdout, always exit code 0. Hook failures must never interrupt a Claude Code session.

### Generated `.claude/settings.json`

Roadie generates this file as part of `generate_all_files`. If `.claude/settings.json` already exists, Roadie merges the `hooks` section without touching any other keys.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "npx roadie-mcp prime --project ."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "npx roadie-mcp observe --tool $TOOL --file $FILE"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "npx roadie-mcp reconcile --project ."
          }
        ]
      }
    ]
  }
}
```

Claude Code injects `$TOOL`, `$FILE`, and `$TRANSCRIPT_PATH` environment variables when it fires hook commands. The `observe` subcommand reads `$TOOL` and `$FILE` to record which tool modified which file.

### Merge Strategy

`.claude/settings.json` is pure JSON with no section ownership markers. The merge algorithm is:

1. If file does not exist → create with the full hooks structure above
2. If file exists and is valid JSON → read existing content, merge `hooks` section (append-only — never overwrite existing hook entries), write atomically
3. If file exists and `hooks` key is absent → add `hooks` key with all three entries, preserve all other keys
4. If file exists and some hooks are already present → **append** Roadie entries only to arrays that don't already contain a Roadie command; do not duplicate
5. If file is invalid JSON → log warning to stderr, skip generation without error

```tsx
async function mergeOrCreateClaudeHooks(
  existingPath: string,
  roadieHooks: ClaudeHooksConfig,
  fs: FileSystemProvider
): Promise<string> {
  let config: Record<string, unknown> = {};

  if (await fs.fileExists(existingPath)) {
    try {
      const raw = await fs.readFile(existingPath);
      config = JSON.parse(raw);
    } catch {
      throw new Error('Existing .claude/settings.json contains invalid JSON. Skipping.');
    }
  }

  // Append-only merge — never clobber existing entries
  if (!config.hooks || typeof config.hooks !== 'object') {
    config.hooks = {};
  }
  const existing = config.hooks as Record<string, unknown[]>;

  for (const [event, entries] of Object.entries(roadieHooks)) {
    if (!existing[event]) {
      existing[event] = entries;
    } else {
      // Only append entries whose command is not already present
      const commands = (existing[event] as Array<{ hooks?: Array<{ command?: string }> }>)
        .flatMap(e => e.hooks ?? [])
        .map(h => h.command);
      for (const entry of entries) {
        const newCmds = (entry as { hooks?: Array<{ command?: string }> }).hooks?.map(h => h.command) ?? [];
        if (!newCmds.some(c => commands.includes(c))) {
          existing[event].push(entry);
        }
      }
    }
  }

  return JSON.stringify(config, null, 2);
}
```

### New CLI Subcommands

These three subcommands are added to `bin/roadie-mcp.ts` and handled without going through the MCP protocol layer:

| Subcommand | Implementation | Notes |
| --- | --- | --- |
| `prime --project <path>` | Loads ProjectModel from SQLite into memory cache | Warms all lazy-loaded caches; writes nothing |
| `observe --tool <name> --file <path>` | Writes edit event directly to `learning_events` table in SQLite | Bypasses MCP server; uses `NodeFileSystemProvider` directly |
| `reconcile --project <path>` | Runs `LearningEngine.reconcile()` on end-of-session data | Updates project model snapshots; commits SQLite WAL |

All subcommands write diagnostic output only to stderr. They always exit with code 0.

### Developer Experience End-to-End

```
# One-time setup (VS Code extension or npm install)
npm install -g roadie-mcp

# In any Claude Code session
$ claude

# Invisible — Claude Code fires hooks automatically:
# [SessionStart]  → npx roadie-mcp prime --project .
# [PostToolUse]   → npx roadie-mcp observe --tool Write --file src/auth.ts
# [PostToolUse]   → npx roadie-mcp observe --tool Edit  --file src/auth.ts
# [Stop]          → npx roadie-mcp reconcile --project .
#
# Developer sees nothing. Roadie silently tracks everything.
```

### How This Differs from `.mcp.json`

| Aspect | `.mcp.json` | `.claude/settings.json` hooks |
| --- | --- | --- |
| Purpose | Passive server registration | Active lifecycle callbacks |
| Trigger | LLM decides to call a tool | Claude Code fires automatically |
| Protocol | MCP JSON-RPC | Direct CLI subprocess |
| Autonomy | Requires explicit LLM instruction | Fully deterministic |
| Context preheat | On first tool call | Before first user turn |
| Edit tracking | Only if LLM calls `observe_*` | After every Edit/Write/MultiEdit |
| End-of-session | Never | Always, via Stop hook |

Both files are generated together by `generate_all_files`. They complement each other — `.mcp.json` exposes the 10 on-demand tools; `.claude/settings.json` hooks provide the invisible autonomy layer.

---

## 3. Gemini CLI Integration

Gemini CLI supports MCP servers via the same `.mcp.json` format or its own settings:

```json
{
  "mcpServers": {
    "roadie": {
      "command": "npx",
      "args": ["roadie-mcp", "--project", "."]
    }
  }
}
```

The experience is identical to Claude Code — Gemini provides the model, Roadie provides the tools.

## 4. Cursor / Windsurf / Other MCP Clients

Any MCP-compatible tool that supports stdio transport can connect using the same configuration. The MCP protocol is the standard interface.

## 5. .mcp.json Generator

Roadie generates `.mcp.json` as part of its file generation catalog.

**New generator: `src/generator/templates/mcp-config.ts`**

```tsx
export function generateMCPConfig(model: ProjectModel): string {
  return JSON.stringify({
    mcpServers: {
      roadie: {
        command: 'npx',
        args: ['roadie-mcp', '--project', '.'],
        env: {}
      }
    }
  }, null, 2);
}
```

**File type:** `mcp-config`

**Output path:** `.mcp.json` (project root)

**Trigger:** `roadie/generate_all_files` or `roadie/generate_file` with `fileType: 'mcp-config'`

### Merge Behavior

Unlike Markdown files, `.mcp.json` is pure JSON and cannot use section ownership markers. Instead:

1. If `.mcp.json` doesn't exist → create with Roadie server entry
2. If `.mcp.json` exists and has no `roadie` key in `mcpServers` → add Roadie entry, preserve all other entries
3. If `.mcp.json` exists and has a `roadie` key → update Roadie entry only, preserve all other entries
4. If `.mcp.json` is invalid JSON → log warning, do not modify

```tsx
async function mergeOrCreateMCPConfig(
  existingPath: string,
  roadieEntry: object,
  fs: FileSystemProvider
): Promise<string> {
  let config: Record<string, unknown> = { mcpServers: {} };

  if (await fs.fileExists(existingPath)) {
    try {
      const raw = await fs.readFile(existingPath);
      config = JSON.parse(raw);
    } catch {
      // Invalid JSON — log warning, don't overwrite
      throw new Error('Existing .mcp.json contains invalid JSON. Skipping.');
    }
  }

  if (!config.mcpServers || typeof config.mcpServers !== 'object') {
    config.mcpServers = {};
  }

  (config.mcpServers as Record<string, unknown>).roadie = roadieEntry;
  return JSON.stringify(config, null, 2);
}
```

## 6. [AGENTS.md](http://AGENTS.md) Enhancement

The existing `AGENTS.md` generator (Phase 1.5) is updated to include an MCP integration section.

**New section added to `agent-definitions.ts` template:**

```markdown
<!-- roadie:start:mcp-integration -->
## MCP Server

This project includes a Roadie MCP server for AI tool integration.

### Quick Start

Add to your `.mcp.json` or tool-specific config:

```

{

"mcpServers": {

"roadie": {

"command": "npx",

"args": ["roadie-mcp", "--project", "."]

}

}

}

```

### Available Tools

| Tool | Description |
| --- | --- |
| `roadie/analyze_project` | Scan project structure, tech stack, patterns |
| `roadie/get_project_context` | Get serialized project context for prompts |
| `roadie/generate_all_files` | Regenerate all AI configuration files |
| `roadie/generate_file` | Generate a specific configuration file |
| `roadie/query_patterns` | Query discovered coding conventions |
| `roadie/query_workflow_history` | View past workflow outcomes |
| `roadie/get_recommendations` | Get actionable improvement suggestions |
| `roadie/rescan_project` | Force fresh project analysis |
| `roadie/run_workflow` | Execute a workflow (requires LLM) |
| `roadie/get_workflow_status` | Check workflow progress |
<!-- roadie:end:mcp-integration -->
```

This section uses standard Roadie ownership markers, so human edits are preserved via the append-below merge strategy.

## 7. npm Package Configuration

### package.json additions

```json
{
  "bin": {
    "roadie-mcp": "./out/bin/roadie-mcp.js"
  },
  "files": [
    "out/"
  ]
}
```

### Publishing Strategy

**Option A (recommended for Phase 2):** The `roadie-mcp` binary is bundled inside the VS Code extension. When someone installs the extension, `npx roadie-mcp` works if they reference the extension's package.

**Option B (future):** Publish `roadie-mcp` as a separate npm package for standalone use without the VS Code extension. This requires extracting core modules into a separate package.

Phase 2 implements Option A. The `tsup.config.ts` produces two entry points:

```tsx
export default defineConfig([
  {
    entry: ['src/extension.ts'],       // VS Code extension
    format: ['cjs'],
    target: 'node20',
    external: ['vscode'],
    outDir: 'out',
  },
  {
    entry: ['bin/roadie-mcp.ts'],      // Standalone MCP server
    format: ['cjs'],
    target: 'node20',
    outDir: 'out/bin',
    // No vscode external — it's not imported
  },
]);
```