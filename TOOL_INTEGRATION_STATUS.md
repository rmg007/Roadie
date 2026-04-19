# Roadie — Tool Integration Status

**Last Updated:** 2026-04-15  
**Current Phase:** 1.5 (Active + Passive Mode)  
**Tool-Specific Integration:** Deferred to Phase 2

---

## Current Architecture (v1.0.0)

Roadie generates **generic context files** that all tools can read, but does NOT generate tool-specific configuration files.

### What's Generated

| File | Primary Consumer | Format | Tool Awareness |
|---|---|---|---|
| `.github/copilot-instructions.md` | GitHub Copilot | Markdown | ✅ Copilot-optimized |
| `AGENTS.md` | AI agents (generic) | Markdown | Generic (works for any agent) |
| `CLAUDE.md` | Claude-compatible tools | Markdown | Generic (no `.claude/settings.json` hooks) |
| `.cursor/rules/project.mdc` | Cursor IDE | Cursor markdown | Cursor-optimized |
| `.cursor/rules/*.mdc` | Cursor IDE (per-dir) | Cursor markdown | Cursor-optimized |
| `.github/instructions/*` | Path-scoped context | Markdown | Generic |
| `.roadie/last-scan.json` | Metadata | JSON | Roadie-specific |

### What's NOT Generated (Deferred to Separate Packages)

| Tool | Handled By | Solution | Impact |
|---|---|---|---|
| **Claude Code** | `roadie-claude-connector` (separate npm package) | External MCP server queries Roadie's data | Full project context via MCP tools; optional hooks |
| **Windsurf** | Tool-specific connector (future) | Separate package provides native Windsurf integration | Falls back to generic `AGENTS.md` until Windsurf connector exists |
| **Gemini CLI** | Tool-specific connector (future) | Separate package provides Gemini variant MCP | Manual config required until connector published |
| **VS Code extensions** | Per-extension configuration | Each extension uses its own config location | Each extension unaware of others (no coordination layer planned) |

---

## How Tools Currently Work

### 1. GitHub Copilot (Optimized ✅)
- **Reads:** `.github/copilot-instructions.md`
- **Benefit:** Full tech stack, commands, patterns injected into Copilot's context
- **Roadie Integration:** VS Code extension is Copilot's native environment
- **Status:** Works seamlessly

### 2. Cursor IDE (Supported ✅)
- **Reads:** `.cursor/rules/project.mdc` + `.cursor/rules/*.mdc`
- **Benefit:** Project-level and per-directory Cursor rules
- **Roadie Integration:** Generates Cursor-specific markdown format
- **Status:** Works, but generic (not tool-aware of Cursor-specific features like repository instructions)

### 3. Claude Code (via Separate MCP Package ✅)
- **Reads:** `CLAUDE.md` + `AGENTS.md` (manual reference)
- **Via MCP:** `roadie-claude-connector` queries `.roadie/last-scan.json` and `.roadie/*.db`
- **Benefit:** Full project intelligence via MCP tools (get_project_context, analyze_project, query_patterns, etc.)
- **Setup:** Manual `.mcp.json` configuration required (transparent, not auto-generated)
- **Status:** Full support via external package; users install `roadie-claude-connector` separately and configure `.mcp.json`
- **Recommendation:** See [roadie-claude-connector](https://github.com/your-org/roadie-claude-connector) for installation

### 4. Windsurf (Unsupported ⚠️)
- **Reads:** `AGENTS.md` (fallback only)
- **Benefit:** Generic agent guidance (not Windsurf-aware)
- **Missing:** Windsurf-specific configuration format (if one exists)
- **Workaround:** Users must manually configure Windsurf or copy context from `AGENTS.md`
- **Status:** Does not auto-integrate

### 5. Other MCP-Compatible Tools (Unsupported ⚠️)
- **Reads:** `AGENTS.md` (fallback only)
- **Missing:** `.mcp.json` configuration and Roadie MCP server implementation
- **Status:** No integration path

### 6. VS Code Extensions (Uncoordinated ⚠️)
- **Reads:** Each extension reads its own config (no Roadie coordination)
- **Issue:** Multiple extensions may conflict or duplicate context
- **Example:** Copilot reads `.github/copilot-instructions.md`, but a custom Cody extension might read a different file
- **Status:** No extension-aware coordination

---

## The Tool Integration Gap

### Problem Statement

Roadie generates files that are **tool-agnostic**, but each tool has different capabilities and configuration formats:

- **GitHub Copilot** expects `.github/copilot-instructions.md` and understands keywords like `tech-stack`, `commands`, `patterns`
- **Claude Code** expects `.mcp.json` (for MCP servers) and `.claude/settings.json` (for hooks, settings, prefs)
- **Cursor** expects `.cursor/rules/` files in Cursor markdown format and understands cursor-specific keywords
- **Windsurf** has its own rules system (exact format TBD)
- **VS Code extensions** have varying config formats and no standard location

### Current Solution (Lowest-Common-Denominator)

Roadie generates **generic Markdown files** (`AGENTS.md`, `CLAUDE.md`) that:
- ✅ Work with any tool that can read Markdown
- ✅ Preserve user edits using HTML comment markers
- ❌ Don't leverage tool-specific features or capabilities
- ❌ Require manual setup for most tools (except Copilot + Cursor)
- ❌ Can't provide automatic context injection or hooks

### Recommended Solution (Separate MCP Package)

**Separation of Concerns:**

Roadie focuses on generating context files. Tool integration is handled by separate packages:

1. **Roadie Extension (v1.0.0+)** — Generates context files only
   - `.github/copilot-instructions.md` (for Copilot)
   - `AGENTS.md`, `CLAUDE.md`, `AGENTS.md` (generic guidance)
   - `.cursor/rules/` (for Cursor IDE)
   - `.roadie/last-scan.json`, `.roadie/*.db` (machine-readable data)

2. **roadie-claude-connector (Separate Package)** — MCP Server for Claude Code
   - Exposes 10+ MCP tools via JSON-RPC protocol
   - `get_project_context` — Returns full project intelligence from Roadie data
   - `analyze_project` — Profiles tech stack, patterns, commands
   - `query_patterns` — Returns coding conventions detected by Roadie
   - Works with or without Roadie extension (falls back to direct scan if extension not installed)

3. **Tool-Specific Connectors (Future)**
   - **Windsurf:** `roadie-windsurf-connector` (separate package, if needed)
   - **Gemini CLI:** `roadie-gemini-connector` (separate package, if needed)
   - **Other tools:** Community-contributed connectors

**Benefits:**
- ✅ Roadie stays focused (context file generation)
- ✅ MCP is independent and maintainable
- ✅ Users choose what to install (context files only, or MCP tools, or both)
- ✅ Lower complexity for Roadie codebase
- ✅ Decoupled evolution (Roadie and MCP versions independent)

---

## What This Means for Users Today

### If you use GitHub Copilot
✅ **Fully optimized.** Roadie works seamlessly.

### If you use Claude Code
✅ **Full integration available** via `roadie-claude-connector` MCP package:
- Install: `npm install -g roadie-claude-connector`
- Configure: Add entry to `.mcp.json` pointing to roadie-mcp
- Benefit: Full project context via MCP tools (get_project_context, analyze_project, etc.)
- **Note:** Requires manual `.mcp.json` setup (transparent, not auto-generated by Roadie)
- **Fallback:** Generic `CLAUDE.md` and `AGENTS.md` also available for manual reference

### If you use Cursor
✅ **Supported.** Roadie generates `.cursor/rules/` files.

### If you use Windsurf
⚠️ **Generic fallback only.** You get `AGENTS.md` but no Windsurf-specific integration.
- **Workaround:** Manually configure Windsurf or copy context from `AGENTS.md`

### If you use other MCP clients (Gemini CLI, custom tools)
⚠️ **Generic fallback only.** You get `AGENTS.md` but no `.mcp.json` registration.
- **Workaround:** Manually add `.mcp.json` entry or copy context files

---

## Roadmap

### v1.0.0 (Current Release)
- ✅ Phase 1 + Phase 1.5 complete and stable
- ✅ Copilot + Cursor optimized
- ✅ Generic CLAUDE.md + AGENTS.md for other tools
- ✅ IDE detection (VS Code, Cursor, Claude Code, Windsurf)
- ✅ MCP placeholders removed (architecture decision: separate package)

### v1.0.0 Status
- ✅ Context file generation complete and stable
- ✅ All Phase 1.5 features working (edit tracking, workflow history, learning database)
- ✅ IDE detection integrated (detector module)
- ℹ️ MCP integration via separate `roadie-claude-connector` package (not in Roadie itself)

### Post-v1.0 (Tool-Specific Connectors)
- ⏳ `roadie-windsurf-connector` (if Windsurf integration specs become available)
- ⏳ `roadie-gemini-connector` (if Gemini integration is requested)
- ⏳ Community-contributed connectors for other tools

---

## For Developers

### Using Roadie with Claude Code

**Option 1: Via MCP Package (Recommended)**

Install `roadie-claude-connector` (separate package):
```bash
npm install -g roadie-claude-connector
```

Configure `.mcp.json`:
```json
{
  "mcpServers": {
    "roadie": {
      "command": "roadie-mcp",
      "args": ["--project", "."]
    }
  }
}
```

**Option 2: Manual Context Reference**

Reference context files in your Claude Code prompts:
```
@files CLAUDE.md AGENTS.md
```

Or copy context into your prompts manually.

### Building a New Tool Connector

To build a connector for a new tool (e.g., Windsurf, Gemini):

1. Read the tool's MCP integration spec (if available)
2. Create a separate npm package (e.g., `roadie-windsurf-connector`)
3. Query Roadie's data files (`.roadie/last-scan.json`, `.roadie/*.db`)
4. Expose tool-specific configuration and integration points
5. See `ROADIE_MCP_API.md` for data schema documentation

**Reference Implementation:** See `roadie-claude-connector` repository for example

---

## Summary Table

| Tool | Current Support | Path to Full Integration | User Effort |
|---|---|---|---|
| GitHub Copilot | ✅ Full (automatic) | Auto-discovery in VS Code | Zero |
| Cursor | ✅ Supported (auto-rules) | `.cursor/rules/` auto-generated | Zero |
| Claude Code | ⚠️ Generic (CLAUDE.md) | Via `roadie-claude-connector` MCP | Minimal (install + `.mcp.json`) |
| Windsurf | ⚠️ Generic fallback | Via future `roadie-windsurf-connector` | Manual (copy `AGENTS.md`) |
| Gemini CLI | ⚠️ Generic fallback | Via future `roadie-gemini-connector` | Manual (copy context files) |
| Other MCP clients | ⚠️ Generic fallback | Via tool-specific connector | Manual (copy context files) |
| VS Code extensions | ⚠️ Uncoordinated | Each extension uses own config | Per-extension setup |

---

## Questions?

- **For feature requests:** GitHub Issues
- **For tool-specific integration:** Create an issue with your tool name and describe your setup
- **To help with Phase 2:** Roadie is open for contributions; see `DEVELOPMENT.md`
