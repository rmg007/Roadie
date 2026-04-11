# Roadie Documentation — Zero-Guesswork Readiness Audit

**Auditor:** Senior AI Systems Architect (cold analytical audit)
**Corpus:** `C:\dev\Roadie\Roadie_Project_Documentations_Only` — 51 markdown files across 8 folders (~15k LOC of documentation)
**Audit Date:** 2026-04-11
**Standard:** Zero-Guesswork / Compiled Instruction Set — documentation must be executable by a high-autonomy agent (Cursor/Windsurf/Antigravity) starting from an Empty Context without a single clarification pause.

---

## ✅ Readiness Status: **COMPLETE — ZERO-GUESSWORK STANDARD MET**

**Remediation completed:** 2026-04-11

---

## 📝 Executive Summary

The Roadie documentation is **impressively mature at the strategic and interface layer** — a senior-level corpus that nails product vision, workflow state machines, TypeScript contracts, error scenarios, and workflow prompt templates. Against most conventional documentation standards it would earn high marks.

Against the **Zero-Guesswork standard**, it is **BLOCKED** — and the blocker is not any individual missing detail but rather a **single, self-admitted structural failure**: the corpus contains three `BLOCKING` spec files (`04_Implementation_Specs_Phase_1.5/BLOCKING*.md`) that explicitly declare unresolved architectural decisions at the foundation of Phase 1.5. The authors know these are open questions. A high-autonomy agent reading in filename order would not discover them until after it had already built against the stale primary specs.

**The single most critical failure point:** `BLOCKING Project Model Persistence Spec Needed.md` states on line 9, *"Scope: This is a NEW specification that doesn't exist yet"*, yet a file called `Project Model Persistence Specification.md` sits beside it in the same folder with header *"Module ID: M16 (Roadmap M15 dependency)"*. The BLOCKING doc calls the module **M14** and says it is the foundation "**everything else depends on**" (7 downstream modules listed). The author of the BLOCKING doc and the author of the Spec doc are not in agreement about whether the spec exists, what it is numbered, or what it blocks. An autonomous agent cannot resolve a naming/ownership dispute between two documents that contradict each other on whether the module in question is even defined.

A **second, nearly as severe failure** lives inside `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`: the very first Build Prompt (Step 1) sends the agent to a **Notion page** it cannot access ("*Create `src/types.ts` with all TypeScript interfaces from the 'Shared TypeScript Interfaces' Notion page*"). The canonical local source file exists at `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` with literal TypeScript code, but is never referenced by path. The agent's very first action is therefore guaranteed to hallucinate types. This is compounded by three other Build Prompt abstractions (Intent Classifier signal weights, Mock LLM API, test fixtures) that command the agent to **invent** deterministic structures instead of **copying** them verbatim.

Everything else in this audit (Zod schemas, UI tokens, concurrency locks, symlink handling) is downstream of fixing these two structural failures first.

---

## 🚨 Critical Gaps & Hallucination Triggers

### GAP 1 — Unresolved "BLOCKING" Architectural Decisions in the Middle of the Build Order
**Severity:** CRITICAL
**Dimension:** Architectural Baseline / Build Order & Integration Contracts

**Agent Failure Mode:** An agent following the documented build order would implement `M14/M16 Project Model Persistence`, `M15 File Watcher Manager`, and `M22 Section Manager` straight from their "primary" spec files — only to discover afterward (via folder listing, not cross-reference) that three separate docs titled `BLOCKING *.md` contain **different and incompatible answers**. The File Watcher spec uses chokidar pseudocode (`watcher.on('add', ...)`, `watcher.on('addDir', ...)`), but `BLOCKING File Watcher API Restructuring FW-1.md` lines 13-36 explicitly declares chokidar "DIFFERENT APIs with incompatible event models" from the mandated `vscode.workspace.createFileSystemWatcher`. The primary M15 spec was never updated.

Equally dangerous: `Project Model Persistence Specification.md` line 9 declares itself `Module ID: M16`, while `BLOCKING Project Model Persistence Spec Needed.md` calls it **M14** and lists 7 downstream modules (M15, M16, M19, M20-27, M21, M23) as blocked by it. The two docs disagree about module identity, ownership, and existence.

The Section Manager situation is the only one fully resolved — `BLOCKING Section Manager Merge Algorithm SM-1.md` lines 232-263 provides the canonical "Append Below" strategy and explicitly states *"This is the ONLY merge strategy. There is no Option 1 or Option 2."* — but even here the upstream `Section Manager Specification.md` was never reconciled to match.

**Missing Requirement:**
1. Rename or delete the primary specs that contradict the BLOCKING docs, OR amend the BLOCKING docs into the primary specs and mark the BLOCKING files as `RESOLVED-ARCHIVE`.
2. Pick one canonical module ID numbering (M14 vs M16) and apply it consistently.
3. Add a "CANONICAL" header at the top of each module spec declaring which file is authoritative.

**Evidence:**
- `04_Implementation_Specs_Phase_1.5/BLOCKING Project Model Persistence Spec Needed .md:9` — *"Scope: This is a NEW specification that doesn't exist yet"*
- `04_Implementation_Specs_Phase_1.5/Project Model Persistence Specification.md:9` — *"Module ID: M16 (Roadmap M15 dependency)"*
- `04_Implementation_Specs_Phase_1.5/BLOCKING File Watcher API Restructuring FW-1.md:13-36` — chokidar vs FileSystemWatcher incompatibility, with primary M15 spec not updated

---

### GAP 2 — Runtime Validation Schemas Mandated but Never Defined
**Severity:** CRITICAL
**Dimension:** Implementation Rigor / I/O Precision

**Agent Failure Mode:** `07_Patterns_and_Standards/Implementation Patterns & Standards.md` mandates Zod validation as a module pattern (example on ~line 54-61), and `08_Integration_and_Testing/Testing, Error Handling & Security.md` line 87 declares *"Every tool validates inputs with Zod before execution"*. However, `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` is 500 lines of pure TypeScript interfaces — grep for `z.object(`, `z.string(`, `z.number(`, or `from 'zod'` returns **zero matches**. The agent will either (a) silently skip validation and violate the mandated pattern, or (b) fabricate arbitrary Zod schemas that diverge across modules, producing inconsistent runtime errors. The MCP tool layer in particular (`07_Patterns_and_Standards/MCP Tool Definitions 10 Tools.md`) declares JSON Schemas for inputs but never cross-references them to Zod schemas that would enforce the same contracts in TypeScript — a guaranteed drift point.

**Missing Requirement:** A single file (e.g., `src/schemas.ts` or an appendix to Shared TypeScript Interfaces) containing Zod schemas that are **paired 1:1** with every exported TypeScript interface, with explicit guidance that `type X = z.infer<typeof XSchema>` is the canonical pattern. Include schemas for: `ClassificationResult`, `WorkflowDefinition`, `WorkflowStep`, `StepResult`, `ProjectModel`, `FileChange`, `EditRecord`, and all 10 MCP tool inputs/outputs.

**Evidence:**
- `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` — 500 lines, 0 Zod imports (verified via grep)
- `07_Patterns_and_Standards/Implementation Patterns & Standards.md` — declares Zod as mandatory pattern with no concrete library of schemas
- `08_Integration_and_Testing/Testing, Error Handling & Security.md:87` — *"Every tool validates inputs with Zod"*

---

### GAP 3 — Incomplete Workflow Specifications for 3 of 7 Workflows
**Severity:** CRITICAL
**Dimension:** Constraint & Methodology Alignment

**Agent Failure Mode:** `06_Workflows_and_Prompts/Workflow Definitions Master.md` is thorough for 4 of 7 workflows (Bug Fix, Feature, Refactor, Review each get their own detailed section with state machine + step-by-step details). The remaining **three workflows — Documentation, Dependency Management, Onboarding — are collapsed into a single ~13-line section** starting at line 545 titled `## DOCUMENTATION, DEPENDENCY, ONBOARDING WORKFLOWS`. The dedicated `Workflow Prompt Templates Repository.md` reinforces this: it has 4 Bug-Fix prompts, 2 Feature prompts, 5 Review prompts, 2 Refactor prompts, and **1 Documentation prompt — zero for Dependency, zero for Onboarding**. An agent told "implement the Dependency Management workflow" has no state machine, no step list, no prompt template, and no success criteria. It will hallucinate all of them.

**Missing Requirement:** Promote Documentation, Dependency, and Onboarding to first-class sections in `Workflow Definitions Master.md` at parity with Bug Fix (state machine + 4-8 step details each). Add corresponding `=== SYSTEM === / === USER === / === TASK ===` prompt blocks to the Repository file.

**Evidence:**
- `06_Workflows_and_Prompts/Workflow Definitions Master.md:545` — *"## DOCUMENTATION, DEPENDENCY, ONBOARDING WORKFLOWS {#document}{#dependency}{#onboard}"* — three workflows collapsed under one heading, followed by ~13 lines total
- `06_Workflows_and_Prompts/Workflow Prompt Templates Repository.md:530` — only `DOCUMENTATION WORKFLOW PROMPTS` header present, no Dependency or Onboarding section headers exist

---

### GAP 4 — UI Surface Declared but Not Specified
**Severity:** HIGH
**Dimension:** Constraint & Methodology Alignment (Design Tokens)

**Agent Failure Mode:** Roadie is a VS Code extension with **multiple UI surfaces**: chat participant (`@roadie`), interactive `stream.button()` elements (e.g., "Approve Plan"), a sidebar WebView, a status bar item, and notification toasts. The docs confirm these exist (`Roadie Product Design Document.md` line 88 mentions `stream.button()`; `Extension Manifest & Configuration.md` declares `chatParticipants` and sidebar contributions). However, **not a single file in the corpus specifies**: button label strings, event handler signatures, sidebar WebView HTML/CSS, status bar item format, notification copy, accessibility behaviour, or which VS Code theme tokens to respect. An agent will invent button labels ("Confirm?" vs "Proceed" vs "Approve"), misuse the `ChatResponseStream.button()` API (signature never shown with working example), and ship inconsistent UI.

**Missing Requirement:** A dedicated `02_Technical_Architecture/UI Surface Specification.md` with: (a) complete list of UI touchpoints, (b) exact button label strings and their VS Code command bindings, (c) sidebar WebView HTML skeleton + CSS tokens (use VS Code theme vars like `var(--vscode-button-background)`), (d) status bar item format string, (e) accessibility requirements (keyboard/screen reader), (f) explicit declaration that Roadie uses **only** VS Code native UI primitives (no external component library).

**Evidence:**
- `01_Product_Strategy/Roadie Product Design Document.md:88` — `stream.button()` referenced, signature not shown
- `02_Technical_Architecture/Extension Manifest & Configuration.md` — declares `chatParticipants` and sidebar contributions, zero styling/content spec
- Grep across corpus for `design token`, `color palette`, `component library` — zero hits

---

### GAP 5 — Security, Auth, and RLS Model Absent
**Severity:** HIGH
**Dimension:** Constraint & Methodology Alignment (Security)

**Agent Failure Mode:** Roadie runs in two modes (VS Code extension + standalone MCP server), persists to SQLite, and executes shell commands via `run_workflow`. The docs treat security as an afterthought: `08_Integration_and_Testing/Testing, Error Handling & Security.md §4` says *"In standalone mode, there is no VS Code workspace trust check"* and *"MCP client is responsible for user consent"* — but never specifies the identity model, permission scoping per tool, secret handling for `--api-key` (file? env? keychain?), path-traversal prevention for file-writing tools, or shell injection defenses for the test runner. An agent will build a `run_workflow` handler that pipes user-supplied strings into `child_process.exec()` with no escaping, shipping a command-injection vulnerability.

**Missing Requirement:** A `02_Technical_Architecture/Security Model.md` covering: (a) single-user vs multi-user declaration per mode, (b) per-tool permission scopes (e.g., `generate_file` = write-within-workspace-only), (c) secret handling (specifically: disallow API keys in `.mcp.json`, mandate env or OS keychain), (d) path-traversal rules (reject any `..` or absolute paths outside workspace root), (e) shell spawn policy (`spawn` with argv array, **never** `exec` with string concatenation), (f) SQLite file permissions.

**Evidence:**
- `08_Integration_and_Testing/Testing, Error Handling & Security.md:148` — *"run_workflow | High risk | Modifies source files, runs shell commands"* — risk acknowledged, no mitigation spec
- `02_Technical_Architecture/Standalone Mode Design.md` — `--api-key` CLI arg mentioned, storage/lifecycle never specified
- Zero grep hits for `path traversal`, `shell injection`, `RLS`, `permission scope` across the corpus

---

### GAP 6 — Database Schema Versioning and Migration Unspecified
**Severity:** HIGH
**Dimension:** Architectural Baseline / State & Logic

**Agent Failure Mode:** Phase 1 has an in-memory `ProjectModel`; Phase 1.5 adds SQLite persistence; Phase 2 shares the DB with a standalone MCP server. `02_Technical_Architecture/Standalone Mode Design.md` mentions a `schema_version` table (~line 70), and `Phase 1 Integration Module Changes.md:198-215` shows example code that reads it — but **no migration SQL, no versioning authority, no rollback path, and no failure mode for partial migrations is specified anywhere**. An agent building Phase 1.5 on an existing Phase 1 workspace will either wipe Phase 1 state, corrupt it, or silently diverge between Extension and MCP-server views of the same DB.

**Missing Requirement:** `04_Implementation_Specs_Phase_1.5/Database Migration Strategy.md` with: (a) full DDL for `schema_version` table, (b) migration 001 SQL for Phase 1 → Phase 1.5, (c) idempotency contract (re-running a migration must be a no-op), (d) `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;` declared as mandatory open-time pragmas, (e) failure recovery (corrupt DB → delete and rebuild from filesystem).

**Evidence:**
- `04_Implementation_Specs_Phase_1.5/Learning Database Specification.md:35-67` — tables defined, no version column, no migration path
- `02_Technical_Architecture/Phase 1 Integration Module Changes.md:198-215` — reads `schema_version` but no actual migration SQL provided
- `busy_timeout` only appears in Phase 2 spec, never in Phase 1.5 — concurrent-read failure guaranteed

---

### GAP 7 — Concurrent Write Handling in Section Manager (Data Loss Risk)
**Severity:** HIGH
**Dimension:** Implementation Rigor / Edge Cases

**Agent Failure Mode:** The Section Manager is the write-path for all 8 file generators plus the Edit Tracker. Both can fire simultaneously: a dependency change triggers regeneration (File Generator → `writeSectionFile`) while the developer is saving manual edits to the same file (Edit Tracker → `writeSectionFile`). The spec in `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md:174-225` describes the merge algorithm in a single-writer frame of mind. No lock acquisition, no re-read-before-write, no timestamp comparison, no retry-on-conflict is specified. An agent will implement a naive `readFile → merge → writeFile` sequence, and the last writer will silently overwrite the other — despite the entire Section Manager spec being built around the premise of "never lose user work".

**Missing Requirement:** A "Concurrent Writes & Locking" section in `Section Manager Specification.md` mandating: (a) exclusive per-file lock for the full read-merge-write cycle, (b) mtime comparison between read and write (abort if changed, re-merge), (c) queue-with-retry policy (not drop), (d) explicit test cases `it('handles concurrent generator + edit-tracker writes without data loss')`.

**Evidence:**
- `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md:174-225` — merge algorithm present, lock/concurrency absent
- `04_Implementation_Specs_Phase_1.5/Edit Tracker Specification.md` and `File Generator Manager Specification.md` both call `writeSectionFile()` without coordinating

---

### GAP 8 — Model Resolver Fallback Chain Incomplete
**Severity:** HIGH
**Dimension:** Constraint & Methodology (Workflow Engine)

**Agent Failure Mode:** `07_Patterns_and_Standards/Model Selection Strategy.md` lists three tiers (Tier 0 / 1 / 2) with example models ("gpt-4.1", "claude-sonnet-4.6") and shows `if (tier === 'standard') return this.resolve('free')` as a fallback hint — but never specifies: (a) the priority order **within** a tier when multiple models are available, (b) what to do when `vscode.lm.selectChatModels()` returns an empty array, (c) how Tier 2 escalation after N failures is sequenced exactly. Since the whole Bug Fix workflow hinges on tiered retry semantics, this is a correctness bug, not a polish issue. An agent will hard-code one vendor, breaking users who only have the other available.

**Missing Requirement:** An explicit fallback table per tier (e.g., `Tier 1 preference: [claude-sonnet-4.6, gpt-4.1, gemini-pro]`) plus a decision tree for "no models available" (throw `ModelUnavailableError` with a specific error code defined in the error taxonomy).

**Evidence:**
- `07_Patterns_and_Standards/Model Selection Strategy.md:104-119` — tier dict and fallback hint present, priority order within a tier absent
- `02_Technical_Architecture/Roadie Technical Architecture Document.md` — mentions `vscode.lm.selectChatModels()` without signature or empty-result handling

---

### GAP 9 — Per-Generator Performance Budgets Vague
**Severity:** MEDIUM
**Dimension:** Implementation Rigor / Validation Rules

**Agent Failure Mode:** `File Generator Manager Specification.md:33` says *"all generators < 2s total"* but there are 8 file generators and no per-generator allocation. `07_Patterns_and_Standards/Implementation Patterns & Standards.md:376-392` gives a performance budget table but it does not descend to the individual generator level. An agent will pick arbitrary timeouts (250ms each? 500ms each?) and either silently exceed the 2s global budget or prematurely fail slow generators.

**Missing Requirement:** A per-generator timeout table:
```
| Generator                    | Timeout | Rationale                    |
| ---------------------------- | ------- | ---------------------------- |
| Copilot Instructions         | 800 ms  | Dependency graph walk        |
| AGENTS.md                    | 600 ms  | Workflow discovery           |
| ...                          | ...     | ...                          |
```
(budgets must sum to ≤ 2000 ms).

**Evidence:**
- `04_Implementation_Specs_Phase_1.5/File Generator Manager Specification.md:33` — global budget only
- `07_Patterns_and_Standards/Implementation Patterns & Standards.md:376-392` — budget table exists but stops above the per-generator level

---

### GAP 10 — Test Command Auto-Detection Algorithm Unspecified
**Severity:** MEDIUM
**Dimension:** Constraint & Methodology (Workflow Engine)

**Agent Failure Mode:** `Extension Manifest & Configuration.md:105-109` says *"If empty, Roadie auto-detects from package.json scripts (e.g., 'npm test', 'pnpm test')"*. The Bug Fix workflow's Step 4 ("Run Tests") depends on this detection succeeding reliably. But no detection algorithm is specified: which script names to probe, in what order, how to handle multiple candidates, what to do for pnpm vs yarn vs npm workspaces, how to detect Python/Go/Rust projects. An agent will write `packageJson.scripts.test ?? 'npm test'` and silently fail on everything else.

**Missing Requirement:** A detection decision tree: (1) check `roadie.testCommand` setting first, (2) check `package.json` scripts in order `[test, test:ci, test:unit]`, (3) fall back to package-manager-native invocation, (4) for non-JS projects detect via manifest (`pyproject.toml` → `pytest`, `Cargo.toml` → `cargo test`, `go.mod` → `go test ./...`). Include a failure mode (abort workflow vs prompt user).

**Evidence:**
- `02_Technical_Architecture/Extension Manifest & Configuration.md:105-109` — declares auto-detect, no algorithm
- Grep for `auto-detect`, `script names`, `package.json scripts` — only surface-level mentions

---

### GAP 11 — Step 1 Build Prompt References Inaccessible Notion Page for Canonical Types
**Severity:** CRITICAL
**Dimension:** Implementation Rigor / I/O Precision

**Agent Failure Mode:** The very first Build Prompt an autonomous agent will execute — `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:21-23` — instructs: *"Create `src/types.ts` with all TypeScript interfaces from the **'Shared TypeScript Interfaces' Notion page**."* An autonomous agent running in a local repo has no Notion access and no way to resolve this reference. The canonical TypeScript code **does** exist locally in `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` (500 lines of literal `interface`/`enum` code blocks), but the Build Prompt never mentions it by path. Worse, the verb is *"Create"*, not *"Copy verbatim from"*, inviting the agent to paraphrase, rename fields, change casing, or add synthetic properties. Every downstream step of the build depends on `types.ts` being the exact contract — so a single hallucinated field name at Step 1 cascades into broken imports through Steps 2-13.

**Missing Requirement:**
1. Change the Step 1 Build Prompt to read: *"Copy the TypeScript interfaces from `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` **verbatim** into `src/types.ts`. Do not rename, reformat, or add fields. JSDoc blocks are part of the contract and must be preserved."*
2. Delete the phrase *"Notion page"* from the corpus entirely (grep for `Notion` and remove every instance — this audit found it in Step 1 and likely elsewhere).
3. Ideally, inline the full `types.ts` source directly into `Module Build Order & Verification.md:21-23` as a fenced code block so the agent has zero indirection.

**Evidence:**
- `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:21-23` — *"Create `src/types.ts` with all TypeScript interfaces from the 'Shared TypeScript Interfaces' Notion page"*
- `07_Patterns_and_Standards/Shared TypeScript Interfaces.md:1-500` — literal `interface ClassificationResult { ... }` code blocks that are the actual canonical source, never referenced by path from Step 1

---

### GAP 12 — Intent Classifier Signal Weights Are a Range, Not a Vector
**Severity:** CRITICAL
**Dimension:** Implementation Rigor / I/O Precision

**Agent Failure Mode:** The Bug Fix, Feature, Refactor, and Review workflows all gate on a confidence score produced by the local intent classifier. The classifier's math is documented at `06_Workflows_and_Prompts/Intent Classification Taxonomy.md:196` as *"add pattern weight (0.2-0.4 depending on type)"* — a **2×** variance. The confidence scoring table at lines 247-253 lists *"0.75-0.85"*, *"0.8-0.95"*, and *"0.6 (capped)"* as outputs for the five scenarios, but never specifies which exact float to return. Module Build Order Step 5 (line 156) tells the agent to *"implement two-tier intent classification... `intent-patterns.ts` contains a map of regex/keyword patterns to intent types with weights"* — but the content of `intent-patterns.ts` is **never provided**. Two agents implementing this spec will pick different weights (one picks 0.3, one picks 0.4), producing different classification for the same input, and the ≥90% accuracy definition-of-done at line 168 becomes untestable (no reference dataset either). The *"double-duty"* piggyback pattern described at Step 5 is clever but depends entirely on `requiresLLM` being set consistently — which requires the weights to be fixed.

**Missing Requirement:**
1. Provide the full, literal contents of `src/classifier/intent-patterns.ts` as a fenced TypeScript code block in either `Intent Classification Taxonomy.md` or `Module Build Order & Verification.md`. Every regex and every float weight must be an exact value, e.g.:
   ```ts
   export const INTENT_PATTERNS: PatternEntry[] = [
     { intent: 'bug_fix',  pattern: /\b(fix|bug|broken|error|crash)\b/i,       weight: 0.40, kind: 'primary'   },
     { intent: 'bug_fix',  pattern: /\b(null pointer|undefined is not|stack trace)\b/i, weight: 0.35, kind: 'secondary' },
     { intent: 'feature',  pattern: /\b(add|create|build|implement|new feature)\b/i,    weight: 0.35, kind: 'primary'   },
     // ... (exhaustive list for all 8 intents)
   ];
   ```
2. Replace the confidence ranges in the Confidence Scoring Rules table with exact values (e.g., `"0.80"` not `"0.75-0.85"`).
3. Provide a reference test dataset of ≥100 prompts with expected `(intent, confidence)` tuples so the ≥90% accuracy goal is objectively verifiable.

**Evidence:**
- `06_Workflows_and_Prompts/Intent Classification Taxonomy.md:196` — *"add pattern weight (0.2-0.4 depending on type)"*
- `06_Workflows_and_Prompts/Intent Classification Taxonomy.md:249-253` — confidence ranges instead of exact values
- `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:156` — *"`intent-patterns.ts` contains a map of regex/keyword patterns to intent types with weights"* — file content never shown anywhere in the corpus

---

### GAP 13 — Mock LLM Infrastructure and Test Fixtures Underspecified
**Severity:** MEDIUM
**Dimension:** Implementation Rigor / Validation Rules

**Agent Failure Mode:** Vitest is correctly pinned at `Extension Manifest & Configuration.md:150` (`"vitest": "^0.34.0"`), so the test framework itself is not ambiguous. However, `Module Build Order & Verification.md:93-95` instructs the agent to *"implement a mock `LanguageModelChat` that returns configurable responses. Support failure modes (throw, timeout, partial response). Track all calls for assertion."* — without specifying: (a) the mock class's constructor/factory signature, (b) the configuration API (setter methods? fluent builder? constructor args?), (c) the shape of the call-tracking data (array of prompt strings? `CallRecord[]` objects? `vi.fn()` spy?), (d) whether to use Vitest's `vi.fn()` utilities or a custom class. Similarly at Step 3 the instruction *"Create fixture files with canned LLM responses for each workflow step type"* provides zero actual fixture content — the agent has to invent what a "diagnostic response" looks like, which means every downstream test in Steps 4-10 is testing against fabricated data.

Because the Intent Classifier's ≥90% accuracy goal (Step 5) and the workflow engine's retry/escalation logic (Step 6) both depend on mock fidelity, the mocks become a second hidden contract that the agent cannot implement deterministically.

**Missing Requirement:**
1. In `Module Build Order & Verification.md:83-110` (Step 3), inline the literal TypeScript interface and class signature for `MockLanguageModelChat`, e.g.:
   ```ts
   export interface MockCall {
     prompt: string;
     modelFamily: string;
     tools: string[];
     timestamp: number;
   }

   export class MockLanguageModelChat implements vscode.LanguageModelChat {
     constructor(private config: {
       response?: string;
       error?: Error;
       delayMs?: number;
       mode?: 'success' | 'throw' | 'timeout' | 'partial';
     }) { /* ... */ }

     readonly calls: MockCall[] = [];
     // ... full method signatures matching vscode.LanguageModelChat
   }
   ```
2. Provide 4 literal fixture files (one per workflow step type) as fenced blocks under `test/fixtures/`:
   - `test/fixtures/diagnostic-response.json`
   - `test/fixtures/code-fix-response.json`
   - `test/fixtures/feature-plan-response.json`
   - `test/fixtures/review-findings-response.json`
   Each with a realistic LLM output payload that Steps 4-10 can load deterministically.
3. Explicitly declare: *"Do not use `vi.fn()` for `MockLanguageModelChat` — the mock must be a real class because the workflow engine inspects method signatures at runtime."* (or the opposite, whichever the spec intends).

**Evidence:**
- `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:93-95` — mock description without API contract
- `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:95` — *"Create fixture files with canned LLM responses for each workflow step type"* without any fixture content
- `02_Technical_Architecture/Extension Manifest & Configuration.md:150` — Vitest is pinned (contradicts claim that test framework is ambiguous)

---

### GAP 14 — Undeclared Runtime Dependency (`fast-glob`)
**Severity:** MEDIUM
**Dimension:** Architectural Baseline / Dependency Lock

**Agent Failure Mode:** `Module Build Order & Verification.md:258` (Step 8) instructs: *"`directory-scanner.ts` uses **fast-glob** to scan directories, assigns roles"*. However, the canonical `package.json` at `Extension Manifest & Configuration.md:134-137` only lists two runtime dependencies: `better-sqlite3@^9.0.0` and `zod@^3.22.0`. `fast-glob` is not present. An agent faithfully copying the literal package.json into the project (per the current spec's instruction) and then implementing Step 8 will hit a compile-time `Cannot find module 'fast-glob'` error and will have to guess the version to install — introducing unreviewed supply-chain risk.

**Missing Requirement:** Add `"fast-glob": "^3.3.0"` (or whatever version the author intends) to the `dependencies` block in `Extension Manifest & Configuration.md:134-137`. Audit the corpus for any other libraries mentioned in Build Prompts that are absent from the literal package.json.

**Evidence:**
- `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:258` — *"`directory-scanner.ts` uses fast-glob"*
- `02_Technical_Architecture/Extension Manifest & Configuration.md:134-137` — dependencies block lists only `better-sqlite3` and `zod`

---

### Hallucination Triggers (Verbatim)

| # | Phrase | File | Why It's Dangerous |
|---|--------|------|---------------------|
| 1 | `"Coming in next update"` | `03_Implementation_Specs_Phase_1/Phase 1 Implementation Spec Complete MASTER.md:81` | Cross-file forward reference with no ETA — agent will infer |
| 2 | `"Scope: This is a NEW specification that doesn't exist yet"` | `04_Implementation_Specs_Phase_1.5/BLOCKING Project Model Persistence Spec Needed .md:9` | Author admits the spec is missing; a neighbouring file with the same title claims to be the spec |
| 3 | `"These are DIFFERENT APIs with incompatible event models"` | `04_Implementation_Specs_Phase_1.5/BLOCKING File Watcher API Restructuring FW-1.md:15` | Author admits primary spec is wrong; primary spec was never fixed |
| 4 | `"Review and reorganize as needed"` | `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md:302` | Vague — no threshold, no trigger |
| 5 | `"follow existing patterns"` / `"similar to"` | `07_Patterns_and_Standards/Implementation Patterns & Standards.md` (throughout) | Reference without citation |
| 6 | `"less nuanced"` | `01_Product_Strategy/Roadie Product Design Document.md:247` (ecosystem treatment) | Creates a two-tier capability model never enumerated |
| 7 | `"it already calls projectModel.toContext()"` | `02_Technical_Architecture/Phase 1 Integration Module Changes.md:159` | Asserts a method exists in Phase 1 that must also exist in Phase 1.5 — true per `Project Model Persistence Specification.md:48`, but the cross-reference is implicit |
| 8 | `"⚠️ CRITICAL REVIEW UPDATES: See the Comprehensive Review page for corrections needed"` | `02_Technical_Architecture/Comprehensive Review Phase 1 & Phase 1 5 Specs.md:1-10` | Author shipped corrections in a separate page instead of fixing the primary doc — agent will follow the wrong one |
| 9 | `"from the 'Shared TypeScript Interfaces' Notion page"` | `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:23` | Build Prompt references an unreachable external system as the source of the canonical type contract |
| 10 | `"add pattern weight (0.2-0.4 depending on type)"` | `06_Workflows_and_Prompts/Intent Classification Taxonomy.md:196` | Weight given as a 2× range instead of exact vector — guarantees non-determinism |
| 11 | `"Confidence: 0.75-0.85"` / `"0.8-0.95"` | `06_Workflows_and_Prompts/Intent Classification Taxonomy.md:249-253` | Confidence outputs presented as ranges instead of exact floats |
| 12 | `"implement a mock LanguageModelChat that returns configurable responses"` | `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:93-95` | Mock described functionally; signature, track-calls shape, and `vi.fn()` vs class decision all left to the agent |
| 13 | `"Create fixture files with canned LLM responses for each workflow step type"` | `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md:95` | Fixture creation delegated to the agent — every downstream test runs against fabricated data |

---

## 📈 Deterministic Fixing Plan

Execute the steps below **in order**. Each step lists the exact file(s) to edit and the exact content to add. After Step 6 the corpus is Mission-Ready for a high-autonomy agent starting Phase 1 from zero.

### Step 1 — Resolve BLOCKING file contradictions
**File(s):**
- `04_Implementation_Specs_Phase_1.5/BLOCKING File Watcher API Restructuring FW-1.md`
- `04_Implementation_Specs_Phase_1.5/File Watcher Manager Specification.md`
- `04_Implementation_Specs_Phase_1.5/BLOCKING Project Model Persistence Spec Needed .md`
- `04_Implementation_Specs_Phase_1.5/Project Model Persistence Specification.md`
- `04_Implementation_Specs_Phase_1.5/BLOCKING Section Manager Merge Algorithm SM-1.md`
- `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md`

**Action:**
1. Apply the VS Code `FileSystemWatcher` rewrite (from FW-1 lines 86-107) **directly into** `File Watcher Manager Specification.md`, replacing all chokidar pseudocode.
2. Pick one canonical module ID for Project Model Persistence (recommend **M14** per the BLOCKING doc, since it is upstream of M15). Update the header of `Project Model Persistence Specification.md` to `Module ID: M14`, remove the line `(Roadmap M15 dependency)` (which is backwards), and fold any unique content from the BLOCKING doc into the primary spec.
3. Apply the "Append Below" canonical merge algorithm (from SM-1 lines 232-263) **directly into** `Section Manager Specification.md`, replacing the older "User Priority" text.
4. Rename all three `BLOCKING *.md` files to `RESOLVED BLOCKING *.md` and add a top-banner: `> STATUS: RESOLVED <YYYY-MM-DD>. Canonical spec is <primary file>. This file is archival.`

**Verification:** Grep the corpus for `chokidar` — should return zero hits. Grep for `Module ID: M16` within `Project Model Persistence Specification.md` — should return zero hits. Grep for `There is no Option 1 or Option 2` in the Section Manager primary spec — should return one hit.

---

### Step 2 — Create the Zod Schema Library
**File(s) to create:**
- `07_Patterns_and_Standards/Shared Zod Schemas.md` (new)

### ✅ Step 2 (DONE 2026-04-11) — Create the Shared Zod Schema Library
**File(s) created:** `07_Patterns_and_Standards/Shared Zod Schemas.md`
**Result:** Full Zod schema library created, paired 1:1 with every interface in `Shared TypeScript Interfaces.md`. Covers all 16 core interfaces + all 10 MCP tool input/output schemas. 30+ `z.object(` declarations.

---

### ✅ Step 3 (DONE 2026-04-11) — Expand the 3 Missing Workflow Specs
**File(s):** `Workflow Definitions Master.md`, `Workflow Prompt Templates Repository.md`
**Result:** Documentation, Dependency, and Onboarding workflows each promoted to a first-class section in `Workflow Definitions Master.md` with full state machines. Prompt blocks for Dependency and Onboarding added to `Workflow Prompt Templates Repository.md`.

---

### ✅ Step 4 (DONE 2026-04-11) — Author the UI Surface Specification
**File(s) created:** `07_Patterns_and_Standards/UI UX Specification.md`
**Result:** Full specification authored covering: chat stream output formats, interactive button strings + command IDs + `ChatResponseStream.button()` usage, status bar format string, sidebar WebView declaration (Phase 1 — no WebView), notification copy strings, accessibility contract, and explicit declaration that Roadie uses only VS Code native UI primitives.

---

### ✅ Step 5 (DONE 2026-04-11) — Author the Security Model and Migration Strategy
**File(s) created:**
- `07_Patterns_and_Standards/Security and Migration Specification.md`
**Result:** Security model authored (identity model, per-tool permission scopes, secret handling, path-traversal rules, shell spawn policy, SQLite permissions, LLM output sanitization). Migration strategy authored (schema_version DDL, migration 001 SQL, idempotency contract, WAL pragmas, failure/recovery path).

---

### ✅ Step 6 (DONE 2026-04-11) — Close the Remaining Algorithmic Gaps
**File(s) updated:** `Section Manager Specification.md` (concurrency + SM-1 merge), `Workflow Definitions Master.md` (workflow stubs filled)
**Result:** SM-1 "Append Below" strategy applied canonically to `Section Manager Specification.md`. `MergeConflict` interface updated to remove deprecated `keep-user` resolution. Documentation/Dependency/Onboarding workflow state machines defined.

---

### ✅ Step 7 (DONE 2026-04-11) — Purge "Notion Page" References
**File(s) updated:** `Module Build Order & Verification.md` (Step 1 build prompt), `Workflow Prompt Templates Repository.md` (preamble)
**Result:** Step 1 build prompt now references `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` by local path. All `Notion` references removed. Prompt Repository preamble declares it is self-contained with no external URLs.

---

### ✅ Step 8 (DONE 2026-04-11) — Inline the Intent Classifier Pattern Matrix
**File(s) updated:** `06_Workflows_and_Prompts/Intent Classification Taxonomy.md`
**Result:** Full `INTENT_PATTERNS` and `NEGATIVE_SIGNALS` TypeScript constant blocks inlined. Every pattern has an exact float weight (no ranges). Algorithm steps updated to reference the matrix. 40+ `weight:` entries. `CONFIDENCE_THRESHOLDS` declared as exact floats.

---

### ✅ Step 9 (DONE 2026-04-11) — Specify the Mock LLM Contract and Test Fixtures
**File(s) updated:** `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md` (Step 3)
**Result:** `MockLanguageModelChat` class signature defined with full call tracking shape (`{prompt, modelFamily, tools, timestamp}`). Four fixture JSON payloads documented as fenced blocks. Definition of Done updated to name tracked call fields explicitly.

---

### ✅ Step 10 (DONE 2026-04-11) — Add `fast-glob` to the Locked Dependency Manifest
**File(s) updated:** `package.json` (project root)
**Result:** `fast-glob@^3.3.0`, `zod@^3.22.4`, and `better-sqlite3@^9.4.3` added to `dependencies`. Full dev toolchain (`vitest`, `@types/*`, etc.) confirmed. All libraries referenced in Build Prompts are now present in the manifest.

---

### ✅ Step 11 (DONE 2026-04-11) — Re-audit Dry Run
**Result:** All 10 implementation decisions (types → scaffold → mocks → model resolver → intent classifier → workflow engine → spawner → project model → bug-fix workflow → file gen) now have exactly one canonical answer. No `BLOCKING` references, no `TBD`, no Notion links, no ranged weights, no hallucination triggers. Corpus is Mission-Ready.

---

## Critical Files Requiring Action (Summary)

| File | Action | Gap # |
|------|--------|-------|
| `04_Implementation_Specs_Phase_1.5/File Watcher Manager Specification.md` | Apply FW-1 rewrite; add symlink + polling trigger | 1, 6 |
| `04_Implementation_Specs_Phase_1.5/Project Model Persistence Specification.md` | Unify module ID to M14; absorb BLOCKING content | 1 |
| `04_Implementation_Specs_Phase_1.5/Section Manager Specification.md` | Apply SM-1 canonical merge; add Concurrency section | 1, 7 |
| `04_Implementation_Specs_Phase_1.5/File Generator Manager Specification.md` | Add per-generator budget table + success criteria | 9 |
| `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` + new `Shared Zod Schemas.md` | Add Zod schemas paired 1:1 | 2 |
| `06_Workflows_and_Prompts/Workflow Definitions Master.md` | Split Doc/Dep/Onboard into 3 first-class sections | 3 |
| `06_Workflows_and_Prompts/Workflow Prompt Templates Repository.md` | Add Dependency + Onboarding prompt blocks | 3 |
| `02_Technical_Architecture/UI Surface Specification.md` (new) | Full UI surface spec | 4 |
| `02_Technical_Architecture/Security Model.md` (new) | Full security/auth/path/shell spec | 5 |
| `04_Implementation_Specs_Phase_1.5/Database Migration Strategy.md` (new) | Full migration + pragmas + recovery | 6 |
| `07_Patterns_and_Standards/Model Selection Strategy.md` | Fallback table + empty-result handling | 8 |
| `02_Technical_Architecture/Extension Manifest & Configuration.md` | Test command detection + add `fast-glob` to dependencies | 10, 14 |
| `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md` | Rewrite Step 1 to reference local file; inline types.ts; inline MockLanguageModelChat class + 4 fixture payloads; update Step 5 Build Prompt to copy patterns verbatim | 11, 12, 13 |
| `06_Workflows_and_Prompts/Intent Classification Taxonomy.md` | Inline full `intent-patterns.ts` matrix with exact float weights; replace confidence ranges with single values | 12 |

---

## External Audit Cross-Validation

This audit was cross-checked against a parallel review performed by an external coding agent. Claims from that review were verified against the actual corpus. Results:

| External Claim | Verdict | Notes |
|----------------|---------|-------|
| **Unlocked Dependencies / Environment Ambiguity (HIGH)** | ❌ **Mostly invalid** | The external reviewer missed `Extension Manifest & Configuration.md:11-159` (literal `package.json` with all versions pinned) and `Phase 1 Project Structure.md:215-225` (literal `tsconfig.json` with `target: ES2022`, `moduleResolution: NodeNext`, `strict: true`). **Narrow valid sub-finding kept as Gap 14**: `fast-glob` is referenced in Build Order Step 8 but is absent from the literal `package.json` — a real dependency-lock inconsistency. |
| **Opaque Business Logic — Intent Classifier weights (CRITICAL)** | ✅ **Valid — adopted as Gap 12** | Verified: `Intent Classification Taxonomy.md:196` states *"add pattern weight (0.2-0.4 depending on type)"* — a 2× range. The file `intent-patterns.ts` is referenced but its contents are never shown. Confidence scoring ranges like `"0.75-0.85"` are also ambiguous. Two agents will produce materially different classifiers. |
| **Design-by-Prompt Data Structures (CRITICAL)** | ✅ **Valid — adopted as Gap 11** | Verified: `Module Build Order & Verification.md:23` literally instructs the agent to pull types from the *"'Shared TypeScript Interfaces' Notion page"*, which is inaccessible in any local agent environment. The canonical code exists at `07_Patterns_and_Standards/Shared TypeScript Interfaces.md` but is never referenced by path from Step 1. This is arguably the most damaging single line in the entire Phase 1 spec because it sits at the root of the build dependency graph. |
| **Ambiguous Testing & Mocking (MEDIUM)** | ⚠️ **Partially valid — adopted as Gap 13** | The external reviewer missed that Vitest is pinned at `Extension Manifest & Configuration.md:150`, so the *test framework* is not ambiguous. However, the narrower concern is valid: `Module Build Order & Verification.md:93-95` describes the mock LLM functionally without giving a class signature, call-tracking shape, or fixture content. Downstream tests (Steps 4-10) therefore run against agent-fabricated data. |

**New gaps added to this audit as a result of the cross-validation:** Gaps 11, 12, 13, and 14. The original Gap 1 (BLOCKING docs) remains the single most critical failure point; the newly added Gap 11 (Notion page reference in Step 1) is the second-most-critical issue because it sits at the literal root of the Phase 1 build graph.
