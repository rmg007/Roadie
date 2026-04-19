# Proposal: Separate MCP for Claude Code Integration

**Status:** Alternative to Phase 2 "build MCP inside Roadie"  
**Complexity:** Lower than integrated approach  
**Coupling:** Loose (decoupled from Roadie extension)

---

## Current Phase 2 Plan (Integrated)

```
roadie-App/
├─ src/
│  ├─ extension.ts (VS Code extension)
│  ├─ mcp/
│  │  ├─ server.ts (MCP server code)
│  │  ├─ tools/
│  │  │  ├─ get-project-context.ts
│  │  │  ├─ analyze-project.ts
│  │  │  └─ ... (10+ tools)
│  │  └─ protocol.ts
│  └─ ...
├─ out/bin/roadie-mcp.js (compiled server)
└─ package.json (includes MCP dependencies)

Usage:
  .mcp.json → "command": "node", "args": ["./out/bin/roadie-mcp.js"]
  Roadie extension generates .mcp.json automatically
  Tight coupling: Roadie extension must exist
```

**Downsides:**
- ❌ Adds MCP complexity to Roadie codebase
- ❌ Roadie extension must generate and manage `.mcp.json`
- ❌ MCP server lifecycle tied to Roadie extension
- ❌ Can't use MCP without Roadie extension installed
- ❌ MCP evolves with Roadie (no independent versioning)

---

## Alternative: Separate MCP Package

```
roadie-claude-connector/ (NEW REPO)
├─ src/
│  ├─ mcp-server.ts (MCP protocol handler)
│  ├─ tools/
│  │  ├─ get-project-context.ts
│  │  ├─ analyze-project.ts
│  │  └─ ... (10+ tools)
│  └─ roadie-client.ts (queries Roadie database or scans)
├─ bin/roadie-mcp.js
├─ package.json
└─ README.md

Usage:
  npm install -g roadie-claude-connector
  # Manual .mcp.json setup:
  {
    "mcpServers": {
      "roadie": {
        "command": "roadie-mcp",
        "args": ["--project", "."]
      }
    }
  }
  No dependency on Roadie VS Code extension
```

**Advantages:**
- ✅ Completely separate codebase
- ✅ Roadie extension stays focused (generate context files only)
- ✅ MCP can be installed/used without Roadie extension
- ✅ Independent versioning and release cycle
- ✅ Users can choose: use MCP, or use generic context files
- ✅ Can be used with Roadie database (if extension installed) OR without (scans codebase)
- ✅ Lower maintenance burden on Roadie team

---

## How They Interact

### Scenario A: Both Installed (Optimal)

```
┌─────────────────────────┐
│   VS Code Window        │
│ ┌─────────────────────┐ │
│ │  Roadie Extension   │ │  Generates:
│ │  - Project scanner  │ │  - .github/copilot-instructions.md
│ │  - File watcher     │ │  - AGENTS.md
│ │  - SQLite DB        │ │  - CLAUDE.md
│ │  - Learning DB      │ │  - .cursor/rules/*
│ └─────────────────────┘ │
└─────────────────────────┘
         ↓ (reads)
    .roadie/last-scan.json
    .roadie/*.db (SQLite)
         ↑
┌─────────────────────────┐
│   Claude Code Session   │
│ ┌─────────────────────┐ │
│ │ roadie-mcp server   │ │  Provides tools:
│ │ (separate process)  │ │  - get_project_context
│ │ - Reads Roadie DB   │ │  - analyze_project
│ │ - Returns tools     │ │  - query_patterns
│ └─────────────────────┘ │  (etc.)
└─────────────────────────┘
```

**Benefits:**
- Extension maintains database
- MCP queries the database
- MCP is fast (doesn't re-scan)
- Both can update independently

### Scenario B: Only MCP Installed (Fallback)

```
┌─────────────────────────┐
│   Claude Code Session   │
│ ┌─────────────────────┐ │
│ │ roadie-mcp server   │ │  Provides tools:
│ │ (separate process)  │ │  - Scans project directly
│ │ - Scans codebase    │ │  - Returns results
│ │ - Returns tools     │ │  (no Roadie DB available)
│ └─────────────────────┘ │
└─────────────────────────┘
```

**Use case:** User doesn't have VS Code, only uses Claude Code

---

## Architecture Comparison

| Aspect | Integrated MCP (Phase 2 Plan) | Separate MCP (Proposed) |
|---|---|---|
| **Codebase** | roadie-App/ (Roadie owns) | roadie-claude-connector/ (separate repo) |
| **Ownership** | Roadie team | Claude integration specialist |
| **Dependencies** | Adds MCP + RPC + node libs to Roadie | Minimal, focused on MCP only |
| **Roadie Extension** | Must generate `.mcp.json` | Optional (standalone works) |
| **Release Cycle** | Tied to Roadie releases | Independent versioning |
| **User Setup** | Automatic (extension does it) | Manual (user edits `.mcp.json`) |
| **Database Access** | Direct (in same process) | Query via file system (`.roadie/*.db`) |
| **Complexity** | Higher (both extension + server) | Lower (server only) |
| **Maintenance** | Roadie team maintains MCP | Separate team maintains MCP |
| **Testing** | Integrated test suite | Separate test suite |

---

## Implementation

### Roadie Extension (v0.7.11+)

**No MCP server code needed.**

Just ensures Roadie data is accessible:
- SQLite databases in `.roadie/`
- JSON scan summary in `.roadie/last-scan.json`
- Generated context files (as today)

**That's it.** Extension stays focused.

### roadie-claude-connector (New Package)

```typescript
// bin/roadie-mcp.ts
import { mcpServer } from './src/mcp-server';
import { RoadieClient } from './src/roadie-client';

const server = mcpServer();
const roadie = new RoadieClient(process.cwd());

server.registerTools([
  {
    name: 'get_project_context',
    description: 'Get full project intelligence from Roadie',
    handler: (args) => roadie.getProjectContext(args),
  },
  {
    name: 'analyze_project',
    description: 'Analyze project structure and tech stack',
    handler: (args) => roadie.analyzeProject(args),
  },
  // ... more tools
]);

server.start();
```

```typescript
// src/roadie-client.ts
export class RoadieClient {
  constructor(private projectRoot: string) {}

  async getProjectContext() {
    // Try to read Roadie's SQLite database (if extension installed)
    try {
      const db = this.openRoadieDatabase();
      return db.queryProjectModel();
    } catch {
      // Fallback: scan codebase directly
      return this.scanProject();
    }
  }

  private async scanProject() {
    // Independent project scanner (no dependency on extension)
    // Same logic as Roadie extension, but standalone
  }
}
```

**Key feature:** Works with or without Roadie extension.

---

## Installation & Usage

### Users with Roadie Extension (Optimal Path)

```bash
# 1. Roadie extension auto-generates context files
#    (happens in VS Code)

# 2. User installs Claude connector
npm install -g roadie-claude-connector

# 3. User sets up .mcp.json
cat > .mcp.json << 'EOF'
{
  "mcpServers": {
    "roadie": {
      "command": "roadie-mcp",
      "args": ["--project", "."]
    }
  }
}
EOF

# 4. Run Claude Code
claude

# MCP server queries Roadie's database (fast, accurate)
```

### Users WITHOUT Roadie Extension

```bash
# 1. Install roadie-claude-connector
npm install -g roadie-claude-connector

# 2. Set up .mcp.json (same as above)

# 3. Run Claude Code
claude

# MCP server scans project directly (no Roadie extension needed)
```

---

## Why This Is Better

1. **Separation of Concerns**
   - Roadie extension: "Generate context files and maintain project model"
   - MCP: "Expose Roadie data via MCP protocol"
   - Each does one thing well

2. **Lower Coupling**
   - MCP doesn't depend on extension
   - Extension doesn't need to know about MCP
   - Can build independently

3. **Simpler Setup**
   - No automatic `.mcp.json` generation (too magical)
   - User explicitly configures (transparent)
   - Familiar pattern (same as other MCPs)

4. **Better for Contributors**
   - MCP is a small, focused package
   - Easy to understand and improve
   - Clear boundaries (doesn't touch VS Code code)

5. **Easier Testing**
   - MCP can be tested standalone
   - Doesn't require VS Code test harness
   - Faster iteration

6. **Enables Alternative Implementations**
   - Someone could build `roadie-windsurf-connector` separately
   - Someone could build `roadie-cursor-connector` separately
   - Ecosystem grows organically

---

## Trade-offs

| Aspect | Pro | Con |
|---|---|---|
| **User Setup** | Users control `.mcp.json` | Not automatic (requires manual config) |
| **Maintenance** | Separate concerns | Need to maintain two repos |
| **Dependencies** | MCP is lightweight | Extension + MCP are separate packages |
| **Documentation** | Clear boundaries | More docs needed |

---

## Recommendation

**Build separate `roadie-claude-connector` MCP.**

Reasons:
1. **Roadie extension stays focused** (generate context files, maintain model)
2. **MCP is minimal and focused** (expose data via protocol)
3. **Users have choice** (use MCP, or use generic context files, or both)
4. **Decoupled evolution** (each can improve independently)
5. **Enables ecosystem** (others can build connectors too)
6. **Cleaner architecture** (follows Unix philosophy)

---

## Phasing

**v0.7.11 (Current):**
- ✅ IDE detection (already done)
- ✅ Document Roadie's data interfaces (what MCP will query)
- ✅ Ensure `.roadie/` directory exports stable JSON/DB schema

**v0.7.12 (Optional, if needed before v1.0):**
- Start `roadie-claude-connector` as separate npm package
- Simple MCP server that reads Roadie data
- Preliminary testing

**v1.0:**
- `roadie-claude-connector` published to npm
- Documentation on how to install and configure
- Example `.mcp.json` in Roadie repo

---

## Questions for You

1. **Does this resonate?** (Separate MCP vs integrated approach)
2. **Should we document the data schema** Roadie exposes (for MCP to query)?
3. **Timeline?** (Build MCP before v1.0, or after stabilizing extension?)
