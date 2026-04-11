# 🚀 Phase 2 Implementation Specification — Master Index

## MCP Server, Standalone Mode, Cross-Tool Integration

**Status:** 📝 READY TO BUILD

**Scope:** MCP Server + Standalone Mode + Cross-Tool Config

**Build on Top Of:** Phase 1 (Active Mode) + Phase 1.5 (Passive Mode)

**Target Audience:** AI Coding Agents (Claude Code, Codex)

**Estimated Build Time:** 15–20 hours

**Milestones:** M21 (Scaffold), M22 (Project Tools), M23 (Workflow + File Gen Tools)

---

## What Phase 2 Adds

### Core Capabilities

**MCP Server**

- stdio-based MCP server using `@modelcontextprotocol/sdk`
- Exposes Roadie's core capabilities as 10 MCP tools
- Spawned by VS Code extension as child process OR runnable standalone
- Any MCP-compatible client can connect (Claude Code, Gemini CLI, Cursor, Windsurf)

**Standalone Mode**

- MCP server runs WITHOUT VS Code extension, directly from CLI
- Entry point: `npx roadie-mcp --project .`
- Enables CI/CD integration, Claude Code, Gemini CLI, any MCP client
- 9 of 10 tools work without LLM; only `run_workflow` needs model access

**Core/Shell Split**

- Provider interfaces abstract VS Code APIs (`ModelProvider`, `ProgressReporter`, `FileSystemProvider`, etc.)
- Extension shell provides VS Code implementations; standalone provides Node.js implementations
- 7 existing modules refactored to accept providers; no public interface changes

**Cross-Tool Configuration**

- `.mcp.json` generator so Claude Code/Gemini CLI auto-discover Roadie
- Enhanced `AGENTS.md` with MCP tool documentation
- Zero-config: generated files include MCP server connection details

### Integration with Phase 1/1.5

- All Phase 1/1.5 modules reused by MCP server via provider abstraction
- Shared SQLite database (WAL mode for concurrent access)
- No breaking changes to any Phase 1/1.5 public interface
- MCP server is optional — extension works perfectly without it

### What Phase 2 Does NOT Include

- ❌ Multi-ecosystem support (still Node.js/TypeScript only)
- ❌ Team/enterprise features
- ❌ MCP Resources or Prompts (only Tools)
- ❌ Streaming workflow progress via MCP notifications (uses polling via `get_workflow_status`)
- ❌ DirectAPIModelProvider implementation (interface only; CI/CD LLM access is opt-in future work)

---

## 📚 Specification Pages

### Architecture & Design

1. **Architecture Overview** — Component diagram, process model, data flow
2. **Core/Shell Split** — Provider interfaces, module refactoring plan
3. **Standalone Mode Design** — CLI entry point, startup sequence, feature degradation
4. **SQLite Concurrent Access** — WAL mode, locking, busy timeout

### MCP Tool Definitions (10 tools)

1. **Project Tools** — `analyze_project`, `get_project_context`, `rescan_project`
2. **Workflow Tools** — `run_workflow`, `get_workflow_status`
3. **Generator Tools** — `generate_file`, `generate_all_files`
4. **Query Tools** — `query_patterns`, `query_workflow_history`, `get_recommendations`

### Integration

1. **Cross-Tool Configuration** — `.mcp.json`, Claude Code setup, [AGENTS.md](http://AGENTS.md) enhancement
2. **Build Order & Dependencies** — 20-step build sequence with verification gates

---

## 🎯 How to Use This Specification

### For AI Coding Agents

**Setup**

1. You've completed Phase 1 + Phase 1.5 ✓
2. Read this Master Index (you're reading it)
3. Read the Architecture Overview page (big picture)
4. Read "Implementation Patterns" from Phase 1 (still applies)

**Build (Steps 1–20)**

1. Start with providers.ts (Step 1)
2. Refactor existing modules to accept providers (Steps 2–8)
3. **GATE: Run all Phase 1/1.5 tests — must pass before proceeding**
4. Build MCP server scaffold (Steps 10–12)
5. Build MCP tools (Steps 13–16)
6. Build cross-tool config generators (Steps 17–18)
7. Integration testing (Step 19)

**Critical Rule:** Step 9 is a hard gate. ALL existing Phase 1/1.5 tests must pass after provider refactoring. Do not write any MCP-specific code until this gate passes.

---

## 📊 Architecture at a Glance

```
Phase 1 (Active) — UNCHANGED
├─ Chat Participant (user prompts)
├─ Intent Classifier
├─ Workflow Engine
└─ Agent Spawner

Phase 1.5 (Passive) — UNCHANGED
├─ File Watcher
├─ Persistent Project Model
├─ File Generators (8 types)
├─ Section Manager
├─ Edit Tracker
└─ Learning Database

Phase 2 (MCP Server) — NEW
├─ Provider Interfaces (abstracts VS Code APIs)
│  ├─ ModelProvider
│  ├─ ProgressReporter
│  ├─ CancellationHandle
│  ├─ FileSystemProvider
│  └─ ConfigProvider
├─ MCP Server (stdio transport)
│  ├─ Tool Registration
│  ├─ Request Routing
│  └─ Error Formatting
├─ MCP Tools (10 tools)
│  ├─ roadie/analyze_project
│  ├─ roadie/get_project_context
│  ├─ roadie/run_workflow
│  ├─ roadie/get_workflow_status
│  ├─ roadie/generate_file
│  ├─ roadie/generate_all_files
│  ├─ roadie/query_patterns
│  ├─ roadie/query_workflow_history
│  ├─ roadie/get_recommendations
│  └─ roadie/rescan_project
├─ Standalone Providers
│  ├─ NullModelProvider (default — client provides LLM)
│  ├─ DirectAPIModelProvider (opt-in for CI/CD)
│  ├─ NodeFileSystemProvider
│  ├─ FileConfigProvider
│  └─ StderrProgressReporter
├─ Extension Integration
│  ├─ MCPManager (child process lifecycle)
│  └─ VSCode Providers (VS Code API wrappers)
└─ Cross-Tool Config
   ├─ .mcp.json generator
   └─ AGENTS.md MCP section
```

---

## 🚨 Critical Design Decisions

### D6: LLM Access in Standalone Mode

**Decision:** Option C — the MCP client provides the LLM. Roadie provides tools, the client provides intelligence.

**Rationale:** Claude Code has Claude. Gemini CLI has Gemini. Cursor has its model. Making Roadie a tool provider (not an LLM consumer) maximizes compatibility. 9 of 10 tools need no LLM at all. `DirectAPIModelProvider` is opt-in for CI/CD.

### D7: MCP Transport

**Decision:** stdio only (no HTTP/SSE).

**Rationale:** stdio is the standard for local tool servers. Claude Code, Gemini CLI, and Cursor all support stdio. HTTP adds complexity (port management, auth) with no benefit for a local tool.

### D8: Workflow Progress Reporting

**Decision:** Polling via `get_workflow_status` (not MCP notifications).

**Rationale:** MCP notifications are not universally supported by all clients. Polling is simpler and works everywhere. Streaming can be added later.

### D9: Feature Workflow Approval in Standalone

**Decision:** Auto-approve plans in standalone mode.

**Rationale:** No interactive UI available. The plan is included in the workflow result. An `autoApprove` option is added for future control.

### D10: Database Concurrent Access

**Decision:** WAL mode + busy_timeout=5000ms.

**Rationale:** Extension and MCP server share one SQLite file. WAL allows concurrent readers + single writer. Contention is rare (extension writes every 5s; MCP writes only on explicit mutations).

---

## 📋 Module Count & Build Time

| Phase | New Files | Modified Files | Est. Lines | Est. Hours | Risk |
| --- | --- | --- | --- | --- | --- |
| Phase 1 | 28 | — | 4,500 | 33.5 | Medium |
| Phase 1.5 | ~30 | ~5 | ~4,000 | 25–30 | High |
| **Phase 2** | **16** | **7** | **~2,300** | **15–20** | **Low–Medium** |
| **Total** | **~74** | **~12** | **~10,800** | **~73.5** | — |

**Phase 2 Risk Drivers:**

- Provider refactoring must not break existing tests (Step 9 gate)
- SQLite concurrent access edge cases
- `run_workflow` via MCP — long-running call, error handling

---

## ⚠️ Backward Compatibility Requirements

**Phase 1/1.5 must work perfectly without Phase 2.** Phase 2 is additive.

### No Breaking Changes

- All `types.ts` interfaces remain unchanged
- Provider refactoring changes internal implementation only
- `container.ts` gets a `RuntimeMode` parameter with default `'extension'`
- If MCP server fails to start, extension logs warning and continues normally

### npm Package

- `roadie-mcp` CLI binary added to `package.json` `bin`
- Does not affect VS Code extension packaging (`.vsix` ignores `bin/`)

---

## ✅ Specification Status

- [x]  Master Index ✅
- [x]  Architecture Overview ✅
- [x]  Core/Shell Split — Provider Interfaces ✅
- [x]  MCP Tool Definitions (10 tools) ✅
- [x]  Standalone Mode Design ✅
- [x]  Cross-Tool Integration ✅
- [x]  Build Order & Dependencies ✅
- [x]  Testing, Error Handling & Security ✅

**All specification pages complete. Ready for implementation.**

---

## 🔗 Quick Links

| Need | Page |
| --- | --- |
| Big picture of Phase 2 | Architecture Overview |
| Provider interfaces (the core/shell split) | Core/Shell Split |
| What each MCP tool does | MCP Tool Definitions |
| Running without VS Code | Standalone Mode Design |
| Setting up Claude Code / Gemini | Cross-Tool Integration |
| What to build first | Build Order & Dependencies |
| How to test | Testing Strategy |

---

---

## ⚡ Review Findings & Additions

During spec review against PDD, TAD, and Roadmap, the following items were identified and addressed:

### Q1: Does WorkflowContext change break types.ts?

**Answer:** Yes — `WorkflowContext.chatResponseStream` and `cancellationToken` fields change from VS Code types to provider interfaces. This is the **one necessary `types.ts` change**. All call sites (Chat Participant, workflow definitions) wrap VS Code types in provider adapters. Covered in Build Order Step 6.

### Q2: How does the extension know the MCP server is ready?

**Answer:** MCPManager watches stderr for `[roadie] MCP server ready`. If not seen within 10 seconds, server is considered failed.

### Q3: What if both extension and MCP server generate files simultaneously?

**Answer:** Last-write-wins via SQLite locks. Section hash comparison prevents duplicate content. Merge-append preserves human edits. Simultaneous generation is unlikely in practice.

### Q4: Should MCP Resources be exposed?

**Answer:** Not in Phase 2. Tools are sufficient. Resources can be added later if clients benefit from subscribing to model changes.

### D11: WorkflowContext Type Change

**Added decision:** `WorkflowContext` updated to use `ProgressReporter` and `CancellationHandle`. Chat Participant wraps VS Code types; MCP server uses standalone implementations.

---

**Created:** April 2026

**Foundation Docs:** PDD v1.0, TAD v1.0 (§2.9), Roadmap v1.0 (M21–M23)

**Next:** Read the detailed spec pages below, then start building at Step 1 (providers.ts).

[🏗️ Phase 2 Architecture Overview](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%8F%97%EF%B8%8F%20Phase%202%20Architecture%20Overview%2033fc821ae63c8146b484ec07d6c752dd.md)

[✂️ Core/Shell Split — Provider Interfaces](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%E2%9C%82%EF%B8%8F%20Core%20Shell%20Split%20%E2%80%94%20Provider%20Interfaces%2033fc821ae63c81af9522e471884cd990.md)

[🔧 MCP Tool Definitions (10 Tools)](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%A7%20MCP%20Tool%20Definitions%20(10%20Tools)%2033fc821ae63c81e78b1eed45871c3849.md)

[🖥️ Standalone Mode Design](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%96%A5%EF%B8%8F%20Standalone%20Mode%20Design%2033fc821ae63c81a8875aebd3e3ea8d5b.md)

[🔗 Cross-Tool Integration & Configuration](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%97%20Cross-Tool%20Integration%20&%20Configuration%2033fc821ae63c811e9ac4cca16f904419.md)

[📊 Build Order & Dependencies](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%93%8A%20Build%20Order%20&%20Dependencies%2033fc821ae63c81479ac8cd2ac6fec4a1.md)

[🧹 Testing, Error Handling & Security](%F0%9F%9A%80%20Phase%202%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%A7%B9%20Testing,%20Error%20Handling%20&%20Security%2033fc821ae63c812b84cce44486666da1.md)