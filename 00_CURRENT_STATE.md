# Roadie — Current State (v1.0.0)
Released: 2026-04-17
Status: Stable (Phase 1 complete, Phase 2 deferred)

**Last Updated:** 2026-04-17  
**Status:** Phase 1 (Active Mode) + Phase 1.5 (Passive Mode) COMPLETE and shipped  
**Build Stage:** Marketplace-ready, v1.0.0 released  

---

## What's Shipped

### Core Features ✅
- **Intent classifier:** 8 intent types (bug fix, feature, refactor, review, document, dependency, onboard, general chat)
- **Workflow engine:** FSM with parallel steps, escalation to higher model tiers on failure, max 3 retries per step
- **7 workflow definitions:** bug fix (8 steps), feature (7), refactor (5), review (5), document (4), dependency (5), onboard (4)
- **Chat interface:** Natural language (`@roadie ...`) + direct slash commands (`/fix`, `/document`, `/review`, `/refactor`, `/onboard`, `/dependency`)
- **Chat variable:** `#roadie` injects full project context into any chat message
- **Code Actions:** Quick-pick actions in editor (Ctrl+.) for symbol-scoped document/review/fix

### Project Intelligence ✅
- **Project model:** Real-time scanner for tech stack, build commands, test runners, directory roles
- **File generation:** Writes 7 artifact families:
  - `.github/copilot-instructions.md` — Copilot context (tech stack, commands, patterns)
  - `AGENTS.md` — Agent-friendly project guide (roles, workflows, directory structure)
  - `CLAUDE.md` — Claude-specific workspace guidance
  - `.cursor/rules/project.mdc` — Cursor IDE rules
  - `.github/instructions/*` — Path-scoped instruction files
  - `.cursor/rules/*.mdc` — Directory-scoped Cursor rules
  - `.roadie/last-scan.json` — Machine-readable scan summary with hashes and write reasons

- **Pattern derivation:** Detects coding conventions from source code (export style, test framework, etc.)
- **Edit tracking:** Preserves user edits to generated files across regenerations using HTML comment markers

### Persistence & Learning ✅
- **SQLite database:** Stores project model, workflow history, learning metrics (optional, graceful fallback if unavailable)
- **Workflow history:** Opt-in logging of every workflow run (success/failure, escalation patterns, context snapshots)
- **Edit tracking:** Opt-in detection of changes to generated files for preference learning
- **Dictionary:** Codebase entity extraction for context queries
- **Scan summary:** `.roadie/last-scan.json` with `writeReason` and `hashPolicy` for reproducibility

### Commands (11 total) ✅
1. **Roadie: Doctor** — Health check (workspace, Copilot, SQLite, generated files, last scan)
2. **Roadie: Get Scan Summary** — Copy last scan JSON to clipboard
3. **Roadie: Run Workflow** — Quick-pick launcher for any workflow
4. **Roadie: Initialize** — Force full project scan + regenerate all files
5. **Roadie: Rescan Project** — Re-scan dependencies and scripts
6. **Roadie: Show Stats** — Display workflow history stats (requires history enabled)
7. **Roadie: Show Last Context** — View most recent LLM context snapshot
8. **Roadie: Show My Stats** — Markdown report with per-intent accuracy, cancellation rates, hot files
9. **Roadie: Reset** — Delete database and reset all state
10. **Roadie: Enable Workflow History** — Start logging workflow runs
11. **Roadie: Disable Workflow History** — Stop logging

### Configuration (7 settings) ✅
| Setting | Type | Default | Purpose |
|---|---|---|---|
| `roadie.modelPreference` | enum | `balanced` | Cost strategy: `economy` (Tier 0 only) / `balanced` (T0→T1) / `quality` (start T1) |
| `roadie.telemetry` | boolean | `false` | Anonymous aggregate telemetry (workflow types, success rates; no code/names) |
| `roadie.editTracking` | boolean | `false` | Track edits to generated files (Phase 1.5) |
| `roadie.workflowHistory` | boolean | `false` | Persist workflow outcomes to SQLite (Phase 1.5) |
| `roadie.autoCommit` | boolean | `false` | Auto-stage generated files in git (Phase 1.5) |
| `roadie.testCommand` | string | `""` | Override auto-detected test command (auto-detects npm/yarn/pnpm test) |
| `roadie.testTimeout` | number | `300` | Max seconds for test suite execution (10–3600) |
| `roadie.contextLensLevel` | enum | `summary` | Output logging: `off` / `summary` / `full` |

---

## Phase 1 vs Phase 2

**Phase 1 shipped v1.0.0** — All Active Mode (chat workflows, intent routing, agent spawning) and Passive Mode (file watcher, persistence, learning database, IDE detection) modules are complete and stable.

**Phase 2 is deferred to v1.1+** — MCP server, standalone mode, and deep cross-tool integration are fully specified but not yet built.

---

## v1.0.0 Changes (2026-04-17)

New modules added in v1.0.0:

- **`detector`** — IDE detection (`detectIDEs()`, `isRunningUnderClaudeCodeHooks()`); detects VS Code, Cursor, Claude Code, Windsurf from env vars and file system markers
- **`api`** — Public API surface; stable exports: `ClassificationResult` (type), `IntentClassifier`, `RoadieError`, `TelemetryReporter`

Expanded modules in v1.0.0:

- **`generator`** — IDE detection integrated with caching; MCP placeholder generation removed (moved to separate package)
- **`shell`** — `RoadieError` and `TelemetryReporter` promoted to public API surface
- **`classifier`** — `IntentClassifier` promoted to public API surface

---

## Module Architecture (v1.0.0)

| Module | Path | Status |
|---|---|---|
| `classifier` | `src/classifier/` | v1.0 stable |
| `analyzer` | `src/analyzer/` | v1.0 stable |
| `generator` | `src/generator/` | v1.0 expanded |
| `shell` | `src/shell/` | v1.0 stable |
| `spawner` | `src/spawner/` | v1.0 stable |
| `engine` | `src/engine/` | v1.0 stable |
| `detector` | `src/detector/` | v1.0 new |
| `api` | `src/api/` | v1.0 new |
| `learning` | `src/learning/` | v1.0 stable |
| `model` | `src/model/` | v1.0 stable |
| `tracking` | `src/tracking/` | v1.0 stable |
| `dictionary` | `src/dictionary/` | v1.0 stable |
| `watcher` | `src/watcher/` | v1.0 stable |

---

## What's NOT Shipped Yet

### Phase 2 (Deferred to Separate Package)
- **MCP Server (roadie-claude-connector):** Moved to separate npm package. Roadie generates data files; external MCP queries them.
- **Cross-tool integration:** Cursor IDE deep linking, native Windsurf support (managed by tool-specific MCPs).

### Architecture Decision
Roadie v1.0.0 focuses on generating context files (CLAUDE.md, AGENTS.md, .cursor/rules/, etc.). Tool-specific integration (MCP for Claude Code, native connectors for other tools) is handled by separate packages that query Roadie's data files (.roadie/last-scan.json, .roadie/*.db).

---

## Tool Integration Status

⚠️ **See [TOOL_INTEGRATION_STATUS.md](TOOL_INTEGRATION_STATUS.md) for full details.**

**TL;DR:** Roadie v1.0.0 is optimized for **GitHub Copilot** (primary), generates generic context files for **Cursor IDE**, and provides generic Markdown fallbacks for other tools. Tool-specific integration (Claude Code hooks, Windsurf native support, MCP servers) is **Phase 2**, deferred to v1.1+.

Currently:
- ✅ **Copilot** — Full integration, auto-context injection
- ✅ **Cursor** — Rule files generated, supported
- ⚠️ **Claude Code** — Generic `CLAUDE.md` + `AGENTS.md` only; no MCP or hooks
- ⚠️ **Windsurf** — Generic `AGENTS.md` fallback only
- ⚠️ **Other MCP clients** — Generic fallback only

---

## Testing Status

- **Unit tests:** 688+ tests, all passing
- **Build:** Compiles to `out/extension.js` via tsup
- **Lint:** ESLint + @typescript-eslint passing
- **Doctor script:** Health check includes workspace validation, generated file verification

---

## Known Limitations

1. **SQLite failure handling:** If better-sqlite3 cannot be compiled (rare), extension falls back to in-memory model with all features available except persistence.
2. **Phase 2 deferred:** MCP and standalone modes not yet built pending Phase 1.5 stabilization.
3. **Copilot Chat integration:** Chat participant echoes `general_chat` intents sometimes instead of routing to LLM. Workaround: use slash commands. Fix pending in v0.5.3 (applies to parallel Roadie VS Code extension, not this app).

---

## Tech Stack

- **Runtime:** Node 22+, VS Code 1.93+
- **Language:** TypeScript 5.2
- **Build:** tsup (CJS output to `out/`)
- **Database:** better-sqlite3 (bundled)
- **Testing:** Vitest 0.34
- **Linting:** ESLint + @typescript-eslint
- **Dependencies:** zod, fast-glob

---

## Deployment

- **Marketplace:** Published as "Roadie" by publisher "roadie"
- **Version:** Synced between `package.json` and git tags (`vX.Y.Z`)
- **Release workflow:** `vsce publish` on tagged commits (automation optional)
- **Icon:** `images/icon.png` included in VSIX
- **VSIX packaging:** `vsce package --no-dependencies`

---

## Version History (Recent)

| Version | Date | Note |
|---|---|---|
| 1.0.0 | 2026-04-17 | **[CURRENT]** IDE detection, public API surface, Phase 1+1.5 fully stable |
| 0.7.10 | 2026-04-15 | Phase 1 & 1.5 complete, marketplace-ready |
| 0.7.8 | 2026-04 | Marketplace listing polish, icon refresh |
| 0.5.0 | 2026-02 | Private testing begins |

### v1.0.0 Changes (2026-04-17)

**Architecture Additions — IDE Detection + Public API:**
- ✅ IDE Detection: `detectIDEs()` in `src/detector/ide-detector.ts` (VS Code, Cursor, Claude Code, Windsurf detection)
- ✅ Hooks Detection: `isRunningUnderClaudeCodeHooks()` — detects Claude Code hook environment
- ✅ Lazy Detection: Integrated into FileGenerator with caching for performance
- ✅ Comprehensive Testing: 14 test scenarios covering all IDE detection paths
- ✅ Remove MCP Placeholders: Phase 2 conditional generation comments removed from `buildFileSpecs()` (MCP moved to separate `roadie-claude-connector` package)
- ✅ Public API Surface: `src/api/index.ts` stabilizes `ClassificationResult`, `IntentClassifier`, `RoadieError`, `TelemetryReporter` as semver-stable exports

---

## Next Steps (Post-Testing)

1. Gather feedback from private testing cohort
2. Fix remaining Phase 1.5 edge cases (e.g., chat participant intent routing)
3. Begin Phase 2 design (MCP server, standalone mode)
4. Public beta release once Phase 1.5 is stable
