# 🧭 START HERE — Roadie Documentation Entry Point

**This is the canonical entry point for any agent or human onboarding to the Roadie specification corpus.**  
**Last verified Mission-Ready:** 2026-04-11 (see `ZERO_GUESSWORK_AUDIT.md`)

> ## ✅ Implementation Status — Updated 2026-04-17
>
> **Phase 1 (Active Mode): COMPLETE** — All 13 modules implemented and tested.  
> **Phase 1.5 (Passive Mode): COMPLETE** — All 7 modules implemented (M15, M16, M19, M21, M22, M23, M24).  
> **v1.0.0 released (2026-04-17)** — IDE detection (`detector` module) and public API surface (`api` module) added. 688+ tests passing.  
> **Phase 2 (MCP Server): NOT YET BUILT** — Fully specified. Deferred to v1.1+.  
>
> The implementation lives at `../roadie-App/` (sibling repo). Do **not** start a new implementation from this corpus — the code already exists and is in stable v1.0.0 release.  
> Agents working on this codebase should focus on **bug fixes, test improvements, and Phase 2 preparation**, not re-implementing Phase 1/1.5.

Roadie is a VS Code extension + standalone MCP server that acts as an "invisible AI workflow engine". This repository contains only documentation — no source code. The implementation repo is `../roadie/`.

---

## 🧰 Installation

For end users and developers who just want to run Roadie, the implementation repo (`C:\dev\Roadie\roadie`) ships with two scripts:

- `scripts/install.js` — plug-and-play installer. Run `npm run install:all` from the repo root to install the VS Code extension and register the Roadie MCP server in Claude Code and Claude Desktop configs. Supports `--skip-extension`, `--skip-mcp`, `--log-level LEVEL`, and `--uninstall`.
- `scripts/doctor.js` — health check. Run `npm run doctor` to verify the build, MCP smoke test, extension registration, and Claude Code MCP entry. Exit 0 if all checks pass.

See `07_Patterns_and_Standards/Installation & Quick Start Guide.md` for the full install/troubleshooting flow.

---

## ⚡ Agent Quick Start

> **Phase 1 and Phase 1.5 are already implemented.** If you are fixing a bug or adding a test, start by reading the relevant spec file and the corresponding source file in `../roadie/src/`. Do not re-implement modules that already exist.

If you are working on **Phase 2** or exploring the codebase from scratch:

1. Read **this file** (you're here).
2. Read `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` (canonical type contracts — matches `../roadie/src/types.ts`).
3. Read `07_Patterns_and_Standards/Shared Zod Schemas.md` (runtime validation schemas — matches `../roadie/src/schemas.ts`).
4. Read `05_Implementation_Specs_Phase_2/Phase 2 Implementation Specification Master In.md` for Phase 2 work.

For historical reference on what was built and how: `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md` documents the original 13-step build plan that produced the current implementation.

Do **not** fetch any external URL, Notion page, or web resource. Do **not** guess at type fields, regex weights, error codes, or merge strategies — every one of those values is literal inline somewhere in this corpus.

---

## 📁 Canonical Reading Order (full)

| Order | Folder / File | Purpose |
|---|---|---|
| 0 | **`Project Mission and Origin Story.md`** | Origin story, project mission, architectural summary, and deferred phase vision |
| 1 | **`00_START_HERE.md`** (this file) | Reading order, canonical file list, ignore list |
| 2 | `01_Product_Strategy/Roadie Product Design Document.md` | Product vision, user scenarios, scope |
| 3 | `01_Product_Strategy/Roadie Development Roadmap.md` | Milestone-by-milestone delivery plan |
| 4 | `02_Technical_Architecture/Roadie Technical Architecture Document.md` | System architecture, module graph, data flows |
| 5 | `02_Technical_Architecture/Extension Manifest & Configuration.md` | `package.json`, VS Code contribution points, settings schema, test-command auto-detection |
| 6 | `02_Technical_Architecture/Standalone Mode Design.md` | MCP server mode, CLI flags, provider adapter pattern |
| 7 | **`07_Patterns_and_Standards/Shared TypeScript Interfaces.md`** | All `src/types.ts` contracts — **COPY VERBATIM, DO NOT PARAPHRASE** |
| 8 | **`07_Patterns_and_Standards/Shared Zod Schemas.md`** | Runtime validation schemas paired 1:1 with interfaces — **COPY VERBATIM** |
| 9 | `07_Patterns_and_Standards/Implementation Patterns & Standards.md` | Coding standards, error handling, performance budgets |
| 9.1 | `07_Patterns_and_Standards/Logging Architecture Specification.md` | `RoadieLogger`, `OutputChannel` lifecycle, log levels, singleton pattern |
| 9.2 | `07_Patterns_and_Standards/Installation & Quick Start Guide.md` | Developer setup (F5), end-user install (.vsix), troubleshooting, workflow history opt-in |
| 10 | `07_Patterns_and_Standards/Model Selection Strategy.md` | Tier hierarchy, `ModelResolver` implementation, `ModelUnavailableError` |
| 11 | `07_Patterns_and_Standards/MCP Tool Definitions 10 Tools.md` | The 10 MCP tools exposed by the server |
| 12 | `07_Patterns_and_Standards/Security and Migration Specification.md` | Security model + DB migration strategy (authoritative for both) |
| 13 | `07_Patterns_and_Standards/UI UX Specification.md` | Every chat stream, button label, status bar state, notification copy |
| 14 | **`06_Workflows_and_Prompts/Intent Classification Taxonomy.md`** | `INTENT_PATTERNS`, `NEGATIVE_SIGNALS`, `CONFIDENCE_THRESHOLDS` — **COPY VERBATIM** |
| 15 | `06_Workflows_and_Prompts/Workflow Definitions Master.md` | State machines for all 7 workflows |
| 16 | `06_Workflows_and_Prompts/Workflow Prompt Templates Repository.md` | `=== SYSTEM === / === USER === / === TASK ===` blocks for every workflow step |
| 17 | **`03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`** | **13-step ordered Build Prompt plan — the primary agent runbook** |
| 18 | `03_Implementation_Specs_Phase_1/Phase 1 Project Structure.md` | Target directory layout, `tsconfig.json`, tool configs |
| 19 | `04_Implementation_Specs_Phase_1.5/Project Model Persistence Specification.md` (M16) | Phase 1.5 SQLite persistence spec |
| 20 | `04_Implementation_Specs_Phase_1.5/File Watcher Manager Specification.md` (M15) | VS Code FileSystemWatcher spec |
| 21 | `04_Implementation_Specs_Phase_1.5/File Generator Manager Specification.md` (M19) | 9 generators, orchestration, per-generator timeout table |
| 22 | `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md` (M22) | Ownership markers + Append Below merge — **the highest-risk module** |
| 23 | `04_Implementation_Specs_Phase_1.5/Edit Tracker Specification.md` (M21) | Stubbed in Phase 1, activated in Phase 1.5 |
| 24 | `04_Implementation_Specs_Phase_1.5/Learning Database Specification.md` | SQLite learning store |
| 24.5 | `04_Implementation_Specs_Phase_1.5/Codebase Dictionary Specification.md` (M24) | Entity extraction, `codebase_entities` SQL schema, `DictionaryGenerator`, prompt injection integration |
| 25 | `05_Implementation_Specs_Phase_2/Phase 2 Implementation Specification Master In.md` | Standalone MCP server phase |
| 26 | `09_Implementation_Specs_Phase_2.5/Phase 2.5 Adaptive Learning and Self-Improvement.md` | Adaptive Learning phase (DESIGN ONLY, DO NOT BUILD UNTIL 2028) |
| 27 | `10_Implementation_Specs_Phase_3/Phase 3 Teams and Enterprise.md` | Teams & Enterprise phase (VISION ONLY, DO NOT BUILD UNTIL 2029) |
| 28 | `08_Integration_and_Testing/End-to-End Integration Scenarios.md` | E2E test scenarios |
| 29 | `08_Integration_and_Testing/Testing, Error Handling & Security.md` | Error taxonomy + security baseline |

---

## 🔖 Canonical Module ID Map

Every module is referenced by these IDs across the corpus. If a stale file references a different number, trust this table.

| ID | Module | Spec File |
|---|---|---|
| M1–M13 | Phase 1 foundation (types, scaffolding, mocks, resolver, classifier, engine, spawner, project model, bug-fix, file-gen, additional workflows, config, marketplace) | `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md` (Steps 1–13) |
| **M15** | File Watcher Manager | `04_Implementation_Specs_Phase_1.5/File Watcher Manager Specification.md` |
| **M16** | Project Model Persistence | `04_Implementation_Specs_Phase_1.5/Project Model Persistence Specification.md` |
| **M19** | File Generator Manager | `04_Implementation_Specs_Phase_1.5/File Generator Manager Specification.md` |
| **M21** | Edit Tracker | `04_Implementation_Specs_Phase_1.5/Edit Tracker Specification.md` |
| **M22** | Section Manager | `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md` |
| **M23** | Learning Database | `04_Implementation_Specs_Phase_1.5/Learning Database Specification.md` |
| **M24** | Codebase Dictionary | `04_Implementation_Specs_Phase_1.5/Codebase Dictionary Specification.md` |
| M25–M32 | 8 file generators (Copilot Instructions, Path Instructions, Agent Definitions, Skills, Hooks, Workflows, Templates, AGENTS.md) | `02_Technical_Architecture/File-Specific Generator Templates All 8.md` |

> **Historical note:** Archived BLOCKING documents at `04_Implementation_Specs_Phase_1.5/ARCHIVED_BLOCKING/` refer to Project Model Persistence as "M14". **That number is obsolete.** The canonical ID is **M16**. Do not implement from any archived BLOCKING file.

---

## 🚫 DO NOT READ — Archived / Historical / Superseded Files

These files exist for historical context only. **A coding agent MUST NOT implement against them.** Every file in this list has a superseded-banner at the top explaining where the canonical content lives.

| Path | Why | Canonical replacement |
|---|---|---|
| `04_Implementation_Specs_Phase_1.5/ARCHIVED_BLOCKING/` (entire folder) | Resolved contradictions | The primary specs (M15, M16, M22) that live one folder up |
| `02_Technical_Architecture/Comprehensive Review Phase 1 & Phase 1 5 Specs.md` | Pre-remediation issue list | `Issues & Fixes Status Tracking Dashboard.md` (now shows all ✅ RESOLVED) |
| `02_Technical_Architecture/Phase 1 5 Architecture Overview_alt.md` | Duplicate of canonical file | `Phase 1 5 Architecture Overview.md` |
| `02_Technical_Architecture/Phase 1 5 Master Review & Execution Guide_alt.md` | Duplicate of canonical file | `Phase 1 5 Master Review & Execution Guide.md` |
| `03_Implementation_Specs_Phase_1/Phase 1 Implementation Specification Master In.md` | Stale outline | `Phase 1 Implementation Spec Complete MASTER.md` (and the Build Order file, which supersedes both) |
| `01_Product_Strategy/PDD Agent Architecture, Project Model & File Gen.md` + `PDD Configuration, Privacy, Scope & Metrics.md` + `Product Design Document PDD v1 0.md` + `Roadie Product Documentation.md` | Fragmented early drafts | `Roadie Product Design Document.md` (consolidated) |
| `02_Technical_Architecture/Additional Issues SHOULD FIX & Nice-to-Haves.md` | Superseded by resolution dashboard | `Issues & Fixes Status Tracking Dashboard.md` |

---

## 📜 Ground Rules for Coding Agents

Read these once — they apply to every file in the corpus.

1. **Local files only.** No file in this corpus references a web URL or Notion page as a build input. If you encounter such a reference, treat it as a hallucination trigger and fall back to the local path listed in this `00_START_HERE.md`.
2. **Copy, don't paraphrase.** Anywhere a spec says "copy verbatim" or gives a fenced TypeScript / JSON / SQL block, reproduce it byte-for-byte. This includes field order, JSDoc comments, regex flags, and whitespace inside strings.
3. **The audit is the contract.** `ZERO_GUESSWORK_AUDIT.md` at the workspace root is the formal readiness audit. Steps 1–11 document the remediation that closed every known gap. If your build hits a gap that audit didn't cover, stop and raise it — don't guess.
4. **Tests are mandatory and literal.** Every spec that says `it('…', ...)` is prescribing a real test case. Implement the described assertions, not approximations of them.
5. **Error taxonomy is closed.** Every thrown error MUST use one of the error codes listed in `08_Integration_and_Testing/Testing, Error Handling & Security.md` or a new code added to that list. Never throw a bare `new Error('…')` for anything a user or test might observe.
6. **Module IDs are binding.** Use the canonical IDs in the map above. When two specs disagree, this file wins.
7. **The build order is ordered.** Do not start Step *n+1* until every Definition of Done box in Step *n* is checked. The order has no implicit parallelism except where a step explicitly permits it.
8. **Prohibited libraries:** `chokidar`, `react`, `electron`, `axios`. The corpus is deliberately dependency-lean. Everything external is already listed in `02_Technical_Architecture/Extension Manifest & Configuration.md`.

---

## ✅ Readiness Checklist

Before handing this corpus to an autonomous agent, confirm:

- [x] `ZERO_GUESSWORK_AUDIT.md` Steps 1–11 all marked `✅ DONE`
- [x] `00_START_HERE.md` (this file) exists and lists canonical module IDs
- [x] `Issues & Fixes Status Tracking Dashboard.md` shows all 22 original issues as ✅ RESOLVED
- [x] No `BLOCKING` files outside `ARCHIVED_BLOCKING/`
- [x] `07_Patterns_and_Standards/Shared Zod Schemas.md` includes all 10 MCP tool input/output schemas
- [x] `06_Workflows_and_Prompts/Intent Classification Taxonomy.md` inlines `INTENT_PATTERNS`, `NEGATIVE_SIGNALS`, and `CONFIDENCE_THRESHOLDS` with exact float values
- [x] `07_Patterns_and_Standards/Model Selection Strategy.md` declares `ModelUnavailableError` and the `MODEL_PRIORITY` table
- [x] `04_Implementation_Specs_Phase_1.5/Codebase Dictionary Specification.md` exists with full SQL schema, 3 module specs (entity-writer, dictionary-query, dictionary-generator), integration points, and Definition of Done
- [x] `04_Implementation_Specs_Phase_1.5/File Generator Manager Specification.md` includes a per-generator budget table with all 9 generators (8 model-change + 1 workflow_complete) summing ≤ 2000 ms for model-change generators
- [x] `02_Technical_Architecture/Extension Manifest & Configuration.md` includes the `roadie.testCommand` auto-detection decision tree
- [x] `07_Patterns_and_Standards/UI UX Specification.md` covers all 7 workflows' HITL buttons, notification copy, sidebar decision, and accessibility contract
- [x] Project root `package.json` matches `Extension Manifest & Configuration.md` versions exactly

When every box is checked, the corpus is Mission-Ready.

> **2026-04-17:** v1.0.0 released. Phase 1 and Phase 1.5 are implemented and stable. IDE detection and public API surface added. The above checklist describes the pre-build readiness state — all boxes remain checked. Begin Phase 2 implementation at `05_Implementation_Specs_Phase_2/Phase 2 Implementation Specification Master In.md`.
