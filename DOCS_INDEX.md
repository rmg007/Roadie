# Roadie Documentation Index

**Last Updated:** 2026-04-17  
**Current App Version:** 1.0.0

---

## Quick Navigation

### For Users
- **README.md** (`roadie-App/`) — Installation, features, commands, settings, usage examples
- **DEVELOPMENT_LOG.md** (`roadie-App/`) — What's implemented, what's not, known issues

### For Developers
- **Current State (v1.0.0)** (`roadie_docs/00_CURRENT_STATE.md`) — Shipped features, deferred work, testing status
- **Tool Integration Status** (`roadie_docs/TOOL_INTEGRATION_STATUS.md`) — **[NEW]** How Roadie handles Copilot, Claude Code, Cursor, Windsurf, and other tools; what's Phase 2
- **File Generation Strategy** (`roadie_docs/FILE_GENERATION_STRATEGY.md`) — **[NEW]** Static file generation (no IDE detection), what files are generated, why, Phase 2 improvements
- **Product Design Document (PDD)** (`roadie_docs/01_Product_Strategy/`) — Vision, narrative, workflow catalog
- **Technical Architecture** (`roadie_docs/02_Technical_Architecture/`) — System design, module contracts
- **Phase 1 Implementation** (`roadie_docs/03_Implementation_Specs_Phase_1/`) — Active Mode modules, build order
- **Phase 1.5 Implementation** (`roadie_docs/04_Implementation_Specs_Phase_1.5/`) — Passive Mode modules, persistence
- **Phase 2 Spec** (`roadie_docs/05_Implementation_Specs_Phase_2/`) — MCP server, tool integration, standalone mode (not yet started)

---

## Feature Reference

### Chat Interface
- **Natural language:** `@roadie fix the login bug` → routes to appropriate workflow
- **Slash commands:** `@roadie /fix`, `/review`, `/document`, `/refactor`, `/onboard`, `/dependency`
- **Context variable:** `#roadie` injects full project context into any chat message

### Workflows (7 total)
| Workflow | Trigger | Steps | Intent |
|---|---|---|---|
| Bug Fix | "fix," "bug," "error," stack traces | 8 | Diagnose → Fix → Verify → Document |
| Feature | "add," "implement," "build" | 7 | Plan → Implement → Test → Document |
| Refactor | "simplify," "clean up," "reorganize" | 5 | Characterize → Refactor → Verify → Test |
| Review | "review," "check," "audit," "security" | 5 | Multi-perspective code review |
| Document | "document," "explain," "comment" | 4 | Generate docs, tests, examples |
| Dependency | "upgrade," "update," "bump" | 5 | Update → Test → Document changes |
| Onboard | "how does," "explain," "learning" | 4 | Guided exploration + documentation |

### Commands (11 total)
See `README.md` Command Palette section for full descriptions.

### Settings (7 total)
| Setting | Type | Default | Purpose |
|---|---|---|---|
| `roadie.modelPreference` | enum | `balanced` | Tier strategy: economy / balanced / quality |
| `roadie.telemetry` | boolean | `false` | Anonymous aggregate stats |
| `roadie.editTracking` | boolean | `false` | Track changes to generated files |
| `roadie.workflowHistory` | boolean | `false` | Persist workflow outcomes |
| `roadie.autoCommit` | boolean | `false` | Auto-commit generated files |
| `roadie.testCommand` | string | `""` | Override auto-detected test runner |
| `roadie.testTimeout` | number | `300` | Max seconds for test execution |
| `roadie.contextLensLevel` | enum | `summary` | Output logging: off / summary / full |

---

## Generated Files

### Copilot Instruction Files
- **`.github/copilot-instructions.md`** — Tech stack, commands, coding patterns
- **`AGENTS.md`** — Project overview, agent roles, workflows, directory structure
- **`CLAUDE.md`** — Claude-specific workspace guidance

### IDE Rule Files
- **`.cursor/rules/project.mdc`** — Project-level Cursor rules
- **`.cursor/rules/*.mdc`** — Directory-scoped Cursor rules
- **`.github/instructions/*`** — Path-scoped instruction files

### Metadata
- **`.roadie/last-scan.json`** — Machine-readable scan summary with hashes, reasons, hashPolicy

---

## Implementation Status

### ✅ Complete (Shipped v1.0.0)
- Phase 1: Active Mode (chat workflows, intent routing, agent spawning)
- Phase 1.5: Passive Mode (file watcher, persistence, learning database)
- All 7 workflows with proper FSM lifecycle
- All 11 commands fully functional
- All 7 configuration settings
- SQLite persistence with graceful fallback
- Edit tracking and pattern derivation
- Code Actions integration

### ⏳ Deferred (Phase 2, pending v1.1+)
- MCP Server implementation
- Standalone mode (non-Copilot environments)
- Cursor IDE deep linking
- Claude Code skill authoring

### 🐛 Known Issues
1. Chat participant sometimes echoes `general_chat` instead of routing to LLM. Workaround: use slash commands.
2. SQLite compilation may fail on rare systems; extension falls back to in-memory mode gracefully.

---

## File Locations

| What | Where |
|---|---|
| Application code | `C:\dev\Roadie\roadie-App/src/` |
| Build output | `C:\dev\Roadie\roadie-App/out/extension.js` |
| Tests | `C:\dev\Roadie\roadie-App/test/` |
| Product docs | `C:\dev\Roadie\roadie_docs/` |
| Spec (read-only) | `C:\dev\Roadie\Roadie_Project_Documentations_Only/` |

---

## Version History

| Version | Date | Status | Notes |
|---|---|---|---|
| 1.0.0 | 2026-04-17 | Current | Phase 1 & 1.5 stable, IDE detection, public API surface |
| 0.7.10 | 2026-04-15 | Released | Phase 1 & 1.5 complete, marketplace-ready |
| 0.7.8 | 2026-04 | Released | Marketplace listing polish |
| 0.5.0 | 2026-02 | Released | Private testing begins |

---

## Next Steps

1. **Gather feedback** from private testing cohort
2. **Fix remaining issues** (chat routing, edge cases)
3. **Plan Phase 2** (MCP, standalone, deep IDE integration)
4. **Public beta** once Phase 1.5 is stable
5. **v1.1 release** with Phase 2 features (MCP, standalone mode)

---

## Document Map (Full)

**Product Strategy**
- `00_CURRENT_STATE.md` — What's shipped, what's deferred, known limitations
- `01_Product_Strategy/Roadie Product Documentation.md` — Doc index
- `01_Product_Strategy/Roadie Product Design Document.md` — Vision + narrative + workflow specs
- `01_Product_Strategy/Key Decisions & Rationale Log.md` — Why major decisions were made
- `01_Product_Strategy/Roadie Development Roadmap.md` — Build sequence and milestones

**Technical Architecture**
- `02_Technical_Architecture/Phase 1 5 Master Review & Execution Guide_alt.md` — TAD overview
- `02_Technical_Architecture/Module-Specific Implementation Guides.md` — Per-module contracts
- `02_Technical_Architecture/Comprehensive Review Phase 1 & Phase 1 5 Specs.md` — Full architecture review

**Implementation Specs**
- `03_Implementation_Specs_Phase_1/Phase 1 Implementation Spec Complete MASTER.md` — Active Mode spec
- `04_Implementation_Specs_Phase_1.5/` — Passive Mode specs (file watcher, persistence, generators)

**Patterns & Standards**
- `07_Patterns_and_Standards/Implementation Patterns & Standards.md` — Code conventions
- `07_Patterns_and_Standards/UI UX Specification.md` — Chat and UI guidelines
- `07_Patterns_and_Standards/Security and Migration Specification.md` — Security practices

**App Documentation**
- `roadie-App/README.md` — Installation, features, usage
- `roadie-App/DEVELOPMENT_LOG.md` — Dev notes, known issues, script reference
