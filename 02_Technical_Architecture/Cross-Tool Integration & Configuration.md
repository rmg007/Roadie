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

## 2. Gemini CLI Integration

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

## 3. Cursor / Windsurf / Other MCP Clients

Any MCP-compatible tool that supports stdio transport can connect using the same configuration. The MCP protocol is the standard interface.

## 4. .mcp.json Generator

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

## 5. [AGENTS.md](http://AGENTS.md) Enhancement

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

## 6. npm Package Configuration

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