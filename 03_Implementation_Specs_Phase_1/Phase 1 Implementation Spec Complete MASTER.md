# 📋 Phase 1 Implementation Spec — Complete (MASTER)

## Complete Navigation & Resource Guide

**Status:** ✅ Ready for AI Agent Implementation  
**Scope:** Active Mode (Chat, Workflows, Intent Classification, Project Model)  
**Target Audience:** AI Coding Agents, Senior Engineers  
**Last Updated:** 2026-04-11

> **Agents:** before reading this file, open `00_START_HERE.md` at the workspace root. That file is the canonical entry point and lists every spec by local path. This file is a subject-matter index; it does NOT replace `00_START_HERE.md`.

---

## 📚 Complete Documentation Set

Every document below is referenced by its local path inside this repository. **No external URLs.** Do not attempt to fetch anything over the network — every required input is on disk.

### **Foundation Documents**

1. **🔧 Shared TypeScript Interfaces** — `07_Patterns_and_Standards/Shared TypeScript Interfaces.md`  
   All 50+ types used across Phase 1.
    - Intent classification types (ClassificationResult, IntentType)
    - Workflow engine types (WorkflowDefinition, WorkflowStep, WorkflowState)
    - Agent spawner types (AgentConfig, AgentResult, AgentRole)
    - Project model types (ProjectModel, TechStackEntry, DirectoryNode)
    - File generation types (GeneratedFileType, GeneratedFile)
    - Error types and VS Code API wrappers
2. **🛡️ Shared Zod Schemas** — `07_Patterns_and_Standards/Shared Zod Schemas.md`  
   Runtime validation schemas paired 1:1 with every interface above, plus schemas for all 10 MCP tool inputs/outputs.
3. **📐 Implementation Patterns & Standards** — `07_Patterns_and_Standards/Implementation Patterns & Standards.md`  
   Rules for AI agents building Phase 1.
    - File length constraints (max 300 lines)
    - Module headers and documentation
    - Type safety & Zod validation
    - Error handling (no silent failures)
    - Async/await patterns (no callbacks)
    - Testing structure and LLM mocking
    - VS Code API usage patterns
    - Performance budgets and limits
    - What to AVOID (circular imports, global state, etc.)
    - Pre-submission checklist

### **Core Specifications**

1. **📊 Module Build Order & Verification** — `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`  
   **This is the primary runbook.** Contains the 13-step ordered Build Prompt plan. Every step has an inlined Build Prompt ready to paste into your IDE, plus verification criteria and Definition of Done.
2. **🔍 Intent Classification Taxonomy** — `06_Workflows_and_Prompts/Intent Classification Taxonomy.md`  
   8 intent types, classification algorithm, and the full `INTENT_PATTERNS` / `NEGATIVE_SIGNALS` / `CONFIDENCE_THRESHOLDS` constants inlined as TypeScript. Also contains the 100-row reference dataset used to measure classifier accuracy.
3. **🔺 Model Selection Strategy** — `07_Patterns_and_Standards/Model Selection Strategy.md`  
   Tier hierarchy, `ModelResolver` implementation, `ModelUnavailableError` taxonomy, `MODEL_PRIORITY` and `TIER_PREFERENCE` maps, and escalation sequencing.

### **Workflow & Behavior Specs**

1. **🌀 End-to-End Integration Scenarios** — `08_Integration_and_Testing/End-to-End Integration Scenarios.md`  
   5 complete workflows from prompt to result.
2. **🌱 Workflow Definitions Master** — `06_Workflows_and_Prompts/Workflow Definitions Master.md`  
   Full state machines for all 7 workflows (Bug Fix, Feature, Refactor, Review, Document, Dependency, Onboarding), each with steps, state transitions, tool scoping, and model tiers.
3. **✍️ Workflow Prompt Templates Repository** — `06_Workflows_and_Prompts/Workflow Prompt Templates Repository.md`  
   `=== SYSTEM === / === USER === / === TASK ===` blocks for every workflow step.

### **Configuration & Structure**

1. **⚙️ Extension Manifest & Configuration** — `02_Technical_Architecture/Extension Manifest & Configuration.md`  
   Complete package.json, VS Code contribution points, settings schema, and the canonical test-command auto-detection decision tree.
2. **📁 Phase 1 Project Structure** — `03_Implementation_Specs_Phase_1/Phase 1 Project Structure.md`  
   Complete file/folder layout, tsconfig.json, vitest config, .gitignore.
3. **🖥️ UI / UX Specification** — `07_Patterns_and_Standards/UI UX Specification.md`  
   Every chat stream format, button label, status bar state, notification copy, and accessibility contract.
4. **🔒 Security and Migration Specification** — `07_Patterns_and_Standards/Security and Migration Specification.md`  
   Security model, shell command allowlist, path-traversal rules, SQLite file permissions, schema migration strategy.

### **MCP and Standalone Mode**

1. **🔧 MCP Tool Definitions (10 Tools)** — `07_Patterns_and_Standards/MCP Tool Definitions 10 Tools.md`  
   Complete specifications for all 10 MCP tools.
2. **🖧 Standalone Mode Design** — `02_Technical_Architecture/Standalone Mode Design.md`  
   CLI flags, provider adapter pattern, API key handling.

---

## 🎯 How to Use This Specification

### **For AI Coding Agents (Claude Code, Cursor, Codex)**

1. **Orientation**
    - Read `00_START_HERE.md` at workspace root (canonical entry point).
    - Read `07_Patterns_and_Standards/Implementation Patterns & Standards.md` (rules).
    - Read `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` (all types you will use).
    - Read `07_Patterns_and_Standards/Shared Zod Schemas.md` (runtime validation contracts).
2. **Build**
    - Open `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`.
    - Execute Step 1, then Step 2, and so on through Step 13.
    - Each step has a Build Prompt — copy it verbatim; do not paraphrase.
    - After each module, run the listed verification commands and tick the Definition of Done boxes.
    - Do not skip steps; dependencies are ordered.
3. **During Implementation**
    - `06_Workflows_and_Prompts/Intent Classification Taxonomy.md` — copy `INTENT_PATTERNS`, `NEGATIVE_SIGNALS`, `CONFIDENCE_THRESHOLDS` verbatim.
    - `07_Patterns_and_Standards/Model Selection Strategy.md` — copy `MODEL_PRIORITY`, `TIER_PREFERENCE`, `ModelResolver`, and `ModelUnavailableError` verbatim.
    - `08_Integration_and_Testing/End-to-End Integration Scenarios.md` — verify behavior matches the scenarios.
    - `03_Implementation_Specs_Phase_1/Phase 1 Project Structure.md` — use for file organization.
4. **Before Submitting**
    - Run the pre-submission checklist in `Implementation Patterns & Standards.md`.
    - Run all tests: `npm run test`
    - Run the linter: `npm run lint`
    - Ensure no file exceeds 300 lines (unless explicitly permitted by a spec).
    - Verify every interface in `src/types.ts` matches `Shared TypeScript Interfaces.md` byte-for-byte.

### **For Code Reviewers**

1. **Module-Level Review**
    - Compare implementation against the relevant Build Prompt in `Module Build Order & Verification.md`.
    - Check that every interface matches `Shared TypeScript Interfaces.md` exactly.
    - Verify tests per the Definition of Done list at the end of each Build Prompt.
2. **Integration Review**
    - Run the 5 scenarios in `End-to-End Integration Scenarios.md` manually.
    - Verify workflows produce expected results.
    - Check error handling uses the canonical `RoadieError` taxonomy (see `Testing, Error Handling & Security.md`).
3. **Quality Review**
    - Ensure code follows `Implementation Patterns & Standards.md`.
    - Check no file exceeds 300 lines.
    - Verify JSDoc headers present on all exported symbols.
    - Confirm no circular imports.

---

## 📋 Document Checklist

Phase 1 Implementation Specification is **COMPLETE** with:

- ✅ Shared TypeScript Interfaces — `07_Patterns_and_Standards/Shared TypeScript Interfaces.md`
- ✅ Shared Zod Schemas (including all 10 MCP tools) — `07_Patterns_and_Standards/Shared Zod Schemas.md`
- ✅ Implementation Patterns & Standards — `07_Patterns_and_Standards/Implementation Patterns & Standards.md`
- ✅ Module Build Order (13 steps with inlined Build Prompts) — `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`
- ✅ Intent Classification Taxonomy (with 100-row reference dataset) — `06_Workflows_and_Prompts/Intent Classification Taxonomy.md`
- ✅ Model Selection Strategy (with `ModelUnavailableError`) — `07_Patterns_and_Standards/Model Selection Strategy.md`
- ✅ End-to-End Scenarios — `08_Integration_and_Testing/End-to-End Integration Scenarios.md`
- ✅ Workflow Definitions Master (all 7 workflows) — `06_Workflows_and_Prompts/Workflow Definitions Master.md`
- ✅ Workflow Prompt Templates Repository — `06_Workflows_and_Prompts/Workflow Prompt Templates Repository.md`
- ✅ Extension Manifest & Configuration (with test-command auto-detection tree) — `02_Technical_Architecture/Extension Manifest & Configuration.md`
- ✅ Phase 1 Project Structure — `03_Implementation_Specs_Phase_1/Phase 1 Project Structure.md`
- ✅ UI / UX Specification — `07_Patterns_and_Standards/UI UX Specification.md`
- ✅ Security and Migration Specification — `07_Patterns_and_Standards/Security and Migration Specification.md`
- ✅ MCP Tool Definitions (10 Tools) — `07_Patterns_and_Standards/MCP Tool Definitions 10 Tools.md`
- ✅ Standalone Mode Design — `02_Technical_Architecture/Standalone Mode Design.md`

---

## 🔗 Quick Links (local paths)

| Need | Go To |
| --- | --- |
| I want to implement a module | `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md` |
| What types exist? | `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` |
| Runtime validation schemas? | `07_Patterns_and_Standards/Shared Zod Schemas.md` |
| Code style rules? | `07_Patterns_and_Standards/Implementation Patterns & Standards.md` |
| How does intent classification work? | `06_Workflows_and_Prompts/Intent Classification Taxonomy.md` |
| Which model tier for which step? | `07_Patterns_and_Standards/Model Selection Strategy.md` |
| Does the workflow really work end-to-end? | `08_Integration_and_Testing/End-to-End Integration Scenarios.md` |
| How to configure the extension? | `02_Technical_Architecture/Extension Manifest & Configuration.md` |
| Where do my files go? | `03_Implementation_Specs_Phase_1/Phase 1 Project Structure.md` |
| What does the UI look like? | `07_Patterns_and_Standards/UI UX Specification.md` |
| Security / migration rules? | `07_Patterns_and_Standards/Security and Migration Specification.md` |
| MCP tool specs? | `07_Patterns_and_Standards/MCP Tool Definitions 10 Tools.md` |
| I'm lost | `00_START_HERE.md` at workspace root |

---

## 📋 Next Steps

1. Open `00_START_HERE.md` at workspace root.
2. Go to `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`.
3. Execute Step 1 (Foundation Types) exactly as written.
4. Iterate — each module depends on prior modules; do not skip steps.
5. Run verification after each module.

---

## 📞 Questions?

If anything is unclear:

1. Check `Implementation Patterns & Standards.md` (most questions answered there).
2. Look at `End-to-End Integration Scenarios.md` (examples of expected behavior).
3. Re-read the specification section relevant to your question — every file is on disk, none require network access.
4. If still stuck, raise the ambiguity rather than guessing. The corpus is supposed to contain exact answers to every build question; a missing answer is a bug to fix, not a gap to fill by inference.

**Remember:** This spec is detailed enough that AI agents should never need to make architectural decisions. If you find yourself debating design, that's a sign the spec needs to be more explicit — surface the ambiguity.

---

**Status:** ✅ Mission-Ready for Phase 1 implementation (as of 2026-04-11)

*This specification is derived from the Roadie Product Design Document, Technical Architecture Document, and Development Roadmap. All cross-references are intentional and documented.*
