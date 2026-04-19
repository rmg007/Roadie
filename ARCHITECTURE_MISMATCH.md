> **Deprecated:** This was a v0.7 → v1.0 decision record. See `00_CURRENT_STATE.md` for current state.

# Architectural Mismatch: VS Code vs Claude Code

**Core Issue:** Roadie is a **VS Code extension**, but being asked to also serve **Claude Code (CLI tool)**. They have fundamentally different architectures.

---

## Runtime Models

### VS Code Extension (Current Roadie Model)

```
User opens VS Code
  ↓
Extension loads (long-lived process)
  ↓
Listens for events: file changes, chat input, commands
  ↓
Reacts to events (stateful, in-memory models)
  ↓
Calls VS Code APIs directly (language models, chat, file system)
  ↓
User gets results (notification, file written, chat response)
  ↓
Extension stays running until VS Code closes
```

**Characteristics:**
- ✅ Long-lived process (persistent state)
- ✅ Direct API access (VS Code LM API, file system APIs)
- ✅ Event-driven (reactive to user actions)
- ✅ Configuration via `package.json` + `.vscode/settings.json`
- ✅ Can manage state, spawn processes, watch files

### Claude Code (Different Model)

```
User runs: claude
  ↓
CLI tool starts, reads `.mcp.json` config
  ↓
Spawns MCP servers (roadie-mcp process)
  ↓
User types a message
  ↓
Claude Code calls MCP tools (JSON-RPC over stdio)
  ↓
Roadie returns results via MCP protocol
  ↓
Claude Code gets LLM response
  ↓
Session ends, both processes terminate
```

**Characteristics:**
- ✅ Stateless CLI (no persistent state between invocations)
- ✅ MCP protocol (JSON-RPC, no direct API access)
- ✅ Hook-based automation (lifecycle events)
- ✅ Configuration via `.mcp.json` + `.claude/settings.json`
- ✅ Can't manage persistent state (each session is independent)

---

## Configuration Paradigm Differences

### VS Code Extension

**Storage:** `package.json` (manifest) + `.vscode/settings.json` (user prefs)

```json
// package.json (declarative)
{
  "contributes": {
    "configuration": {
      "properties": {
        "roadie.modelPreference": {
          "type": "string",
          "enum": ["economy", "balanced", "quality"],
          "default": "balanced"
        }
      }
    }
  }
}

// .vscode/settings.json (user can toggle)
{
  "roadie.modelPreference": "quality"
}
```

**How it works:**
1. User opens settings UI (`Ctrl+,`)
2. Searches "roadie"
3. Sees dropdown, selects preference
4. Extension reads `context.workspaceState.get('roadie.modelPreference')`
5. Behavior changes immediately

### Claude Code

**Storage:** `.mcp.json` (server registration) + `.claude/settings.json` (hooks)

```json
// .mcp.json (register MCP server)
{
  "mcpServers": {
    "roadie": {
      "command": "npx",
      "args": ["roadie-mcp", "--project", "."],
      "env": {}
    }
  }
}

// .claude/settings.json (register hooks)
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
    ]
  }
}
```

**How it works:**
1. User manually edits `.mcp.json`
2. Restarts Claude Code session
3. Claude Code reads config, spawns `roadie-mcp` subprocess
4. Subprocess runs with new env vars
5. Behavior changes on next session start

---

## Key Differences Table

| Aspect | VS Code Extension | Claude Code |
|---|---|---|
| **Process Lifetime** | Long-lived (app lifetime) | Stateless (session lifetime) |
| **State Management** | In-memory + VSCode storage | File system only (no memory) |
| **API Access** | Direct VS Code APIs | MCP protocol (JSON-RPC) |
| **Configuration Method** | UI dropdown + JSON file | Manual JSON file editing + CLI args |
| **Configuration Persistence** | Automatic (VS Code manages) | Manual (user edits files) |
| **LLM Integration** | vscode.lm (Copilot, Claude, etc.) | Provider-agnostic (Claude, Gemini, etc.) |
| **File System** | Direct `fs` API access | Via MCP `read_file`, `write_file` tools |
| **Testing** | Run `npm test`, extension test host | Run `claude` command in shell |
| **Improvement Mechanism** | User installs updated extension | User updates `.mcp.json`, `.claude/settings.json` |
| **Scalability** | Single instance per VS Code | New instance per `claude` invocation |

---

## Why This Matters for Roadie

### Current Problem

Roadie tries to optimize for **both models simultaneously**:

```typescript
// src/extension.ts (VS Code extension code)
export function activate(context: vscode.ExtensionContext) {
  const extension = new RoadieExtension(context);
  extension.init(); // Works great in VS Code
}

// ALSO needs to support...
// $ roadie-mcp --project . (Claude Code MCP server)
// $ roadie init (CLI tool)
// Requires completely different architecture
```

**The tension:**

1. **VS Code mode** (extension):
   - Has stateful models, file watchers, learning database
   - Reads/writes config to `.vscode/settings.json`
   - Manages long-lived processes
   - Can call Copilot directly

2. **Claude Code mode** (MCP server):
   - Must be stateless (CLI subprocess)
   - Reads/writes config to `.mcp.json`, `.claude/settings.json`
   - Receives tool calls via JSON-RPC
   - Returns results as JSON

**These are incompatible without significant refactoring.**

---

## Phase 2 Problem Statement

### The Challenge

To properly support Claude Code, Roadie would need:

1. **Separate entry point:** `roadie-mcp` CLI that:
   - Reads `.mcp.json` on startup
   - Boots project model from SQLite
   - Listens for JSON-RPC tool calls
   - Returns results (no UI, no chat integration)
   - Exits when session ends

2. **Separate configuration:** `.claude/settings.json` with:
   - Hook definitions (SessionStart, PostToolUse, Stop)
   - Not manageable via VS Code settings UI
   - Manual file edits required

3. **Separate improvement mechanism:**
   - User updates `.claude/settings.json`
   - User edits `.mcp.json`
   - No "click Install Extension" button

### Current Roadie Approach

Roadie generates **generic Markdown files** that work with any tool:

```
.github/copilot-instructions.md     ← Copilot reads this
AGENTS.md                           ← Any agent can use this
CLAUDE.md                           ← Claude reads this (manually)
.cursor/rules/project.mdc           ← Cursor reads this
```

**Advantage:** Works with every tool  
**Disadvantage:** Optimizes for none

---

## What This Means

### For VS Code Extension (Current v0.7.10)

✅ Works perfectly
- Settings UI is intuitive
- Long-lived state management is natural
- Copilot integration is seamless
- File watching and learning database work great

### For Claude Code (Deferred to Phase 2)

⚠️ Would require:
1. **New CLI entry point** (`roadie-mcp` command)
2. **New configuration format** (`.mcp.json` registration)
3. **New runtime model** (stateless MCP server)
4. **Different settings mechanism** (no UI, file-based)
5. **Hook system** (lifecycle callbacks)

### For Other Tools (Windsurf, Gemini, etc.)

⚠️ Same situation as Claude Code
- Each needs its own configuration format
- Each needs its own integration point
- Generic Markdown fallback is best we can do today

---

## Strategic Decision Required

### Option A: Keep VS Code-First (Current)
- ✅ V0.7.10 works great
- ✅ IDE detection helps users
- ⚠️ Claude Code / Windsurf / others get generic Markdown
- ⏳ Phase 2 would add parallel CLI tools

### Option B: Refactor for Multi-Tool (Major Change)
- ✅ Would optimize for all tools equally
- ✅ Each tool gets native integration
- ⚠️ Would require significant rewrite
- ⚠️ Would delay v1.0 release significantly

### Option C: Hybrid (Phase 2 Plan)
- ✅ Keep VS Code extension as-is (works)
- ✅ Add `roadie-mcp` CLI server for Claude Code
- ✅ Keep generic files as fallback for others
- ✅ Allow each tool to use native integration if it chooses

---

## Current Plan (Phase 2, Option C)

```
v0.7.11 (current):
  └─ Add IDE detection
  └─ Prepare placeholders for conditional generation

v1.0 (Phase 2):
  ├─ roadie-mcp server (new CLI entry point)
  │  ├─ MCP protocol support
  │  ├─ Tool registry (get_project_context, etc.)
  │  └─ Hook system (.claude/settings.json)
  │
  ├─ Conditional file generation
  │  ├─ When Claude Code detected → generate .mcp.json
  │  ├─ When Windsurf detected → generate windsurf-rules
  │  └─ Generic files always generated as fallback
  │
  └─ Tool-specific improvements
     ├─ Cursor IDE deep rules
     ├─ Claude Code hooks
     └─ Windsurf native integration

v2.0+ (Future):
  └─ Evaluate tool-specific extensions
```

---

## Conclusion

**Yes, the architectures are very different.** This is why:

1. ✅ IDE detection helps (knows what tools are present)
2. ⏳ Phase 2 is necessary (can't do "one-size-fits-all" well)
3. ⚠️ Refactoring required (separate CLI, different config formats)
4. 📋 Phase 2 infrastructure is planned (MCP server, hooks, tool-specific generators)

The current approach (generic Markdown files) is a **pragmatic interim solution** that works but doesn't optimize for any tool's specific capabilities.

Should we proceed with Phase 2 infrastructure in v0.7.11, or keep focus on stabilizing v0.7.10 first?
