# Roadie — Development Roadmap

**CONFIDENTIAL** · Version 1.0 — April 2026

> ## ✅ Implementation Status — 2026-04-17
>
> **Phase 1 (Active Mode): COMPLETE** — All milestones 0–13 implemented.  
> **Phase 1.5 (Passive Mode): COMPLETE** — All milestones implemented (M15, M16, M19, M21, M22, M23, M24).  
> **v1.0.0 shipped (2026-04-17)** — Both phases are live in `../roadie-App/` and stable. IDE detection (`detector` module) and public API surface (`api` module) added.  
> **Phase 2 (MCP Server):** Fully specified, not yet built. Deferred to v1.1+.

---

## How to Read This Document

This roadmap sequences the construction of Roadie across three phases: Phase 1 (Active Mode), Phase 1.5 (Passive Mode), and Phase 2 (MCP Server). Each phase is broken into milestones. Each milestone is scoped to be achievable in a focused AI agent session (1–3 days of directing an agent) and produces a build-green, independently testable increment. Every milestone delivers user-facing value or foundational infrastructure that the next milestone depends on.

All code is written by AI agents. Module specifications are detailed enough for an agent to implement without ambiguity. TypeScript interfaces define all module boundaries. Test cases accompany every module.

---

## v0.1, v0.5, and v1.0 Definitions

### v0.1 — "The First Proof"

**What it is:** The absolute minimum that demonstrates the core value proposition — a single workflow (bug fix) working end-to-end through a VS Code Chat Participant, with a lazy project model providing context.

**What a developer experiences when they install v0.1:**

1. They install the extension. A status bar item shows "Roadie active."
2. They open the Copilot chat panel, select "Roadie" from the agent dropdown.
3. They type "The login page is throwing a 500 error after the last deploy."
4. Roadie classifies this as a bug-fix intent, executes a multi-step workflow — locating the error source, diagnosing, generating a fix, running tests, scanning for siblings, adding a regression test — and streams progress and results back to the chat.
5. If the fix fails tests, Roadie retries with a more capable model (escalation).
6. A `.github/copilot-instructions.md` file appears with basic tech stack context detected during the workflow.

**What is explicitly NOT in v0.1:**

- Feature development, refactoring, code review, documentation, dependency, or onboarding workflows
- Passive mode (file watching, automatic regeneration)
- Sidebar UI
- Edit tracking, workflow history, preference learning
- Multi-ecosystem support (Node.js/TypeScript only)
- Marketplace publishing
- "No-tests" fallback handling

### v0.5 — "Worth Showing"

**What it is:** The first version worth demonstrating to other developers. Four core workflows (bug fix, feature development, refactoring, code review) are working end-to-end. The project model is populated and generates useful AI configuration files. The intent classifier routes prompts reliably.

**Feature set:**

- Bug fix workflow (with no-tests fallback)
- Feature development workflow (with plan approval)
- Refactoring workflow (with characterization tests and incremental safety)
- Code review workflow (5-pass parallel review)
- Intent classifier (two-tier: local + LLM fallback)
- Lazy project model (Node.js/TypeScript ecosystem)
- File generation: `copilot-instructions.md`, `AGENTS.md`, per-language instructions
- Model escalation across all workflows
- Passthrough mode for general questions (enriched with project context)

**Stability expectations:** Workflows complete successfully ≥80% of the time on Node.js/TypeScript projects with tests. Intent classification is correct ≥85% of the time.

**Known limitations:** No passive mode, no file watching, no learning. Node.js/TypeScript only. No sidebar. No marketplace listing yet.

### v1.0 — "Product Complete for Solo Developers"

**What it is:** The full product as described in the PDD. All seven workflows, passive mode with file watching and incremental model updates, learning database, section ownership with edit preservation, all file generation, marketplace publishing as Preview.

**What "done" looks like:**

- All seven workflows operational: bug fix, feature development, refactoring, code review, documentation, dependency management, onboarding
- Passive mode: VS Code FileSystemWatcher monitors dependency files and config; project model updates incrementally; generated files regenerate when the model changes
- Learning database: edit tracking (opt-in), workflow history (opt-in), pattern persistence
- File generation: full catalog (`copilot-instructions.md`, path-specific instructions, agent definitions, skill definitions, hooks, PR templates, issue templates, `AGENTS.md`)
- Section ownership model: Roadie markers, human edit detection, merge-not-overwrite
- Status bar showing workflow state
- Configuration model (all settings in `.vscode/settings.json`)
- Privacy model fully implemented (local-first, zero transmission for core)
- Marketplace published as Preview/Pre-release
- Node.js/TypeScript ecosystem (other ecosystems are post-v1.0)

---

# Phase 1 Milestones — Active Mode

## Milestone 0: Project Scaffolding — ✅ COMPLETE

**What is built:** Full project skeleton — build chain, test harness, extension manifest, minimal Chat Participant.

**Specific modules:**

- `extension.ts` — activate/deactivate entry points
- `chat-participant.ts` — registers `@roadie` Chat Participant, echoes input
- `status-bar.ts` — status bar item showing "Roadie active"
- `types.ts` — shared type definitions (empty shell with core enums)
- `container.ts` — dependency injection container (empty shell)

**Specific files and config:**

- `package.json` with extension manifest: `chatParticipants` contribution point, activation events (`onChat:roadie`)
- `tsconfig.json` targeting ES2022, module NodeNext
- `tsup.config.ts` for bundling
- `vitest.config.ts` for unit tests
- `.vscode/launch.json` for Extension Development Host debugging (F5)
- `@vscode/test-cli` + `@vscode/test-electron` config for integration tests
- `.gitignore`, `.eslintrc.json`, `.prettierrc`
- `AGENTS.md` — project architecture description for AI agents

**What works at the end:**

- `npm run build` produces a bundled `.js` file
- `npm run test` runs Vitest and passes
- F5 launches the Extension Development Host
- `@roadie` appears in the Copilot chat agent dropdown
- Typing a message to `@roadie` echoes it back
- Status bar shows "Roadie active"

**Validation criteria:**

1. `npm run build` exits 0 with no errors
2. `npm run test` exits 0 (at least one trivial test passes)
3. `npm run lint` exits 0
4. Extension loads in the Extension Development Host without errors
5. `@roadie` is visible in the chat agent dropdown
6. Sending "hello" to `@roadie` returns "hello" in the chat response stream
7. Status bar item displays "Roadie active"
8. A test button rendered via `stream.button()` triggers a registered VS Code command when clicked, and the command's output is observable in the test. This validates the button → command round-trip before it becomes a workflow dependency in M9.

> **Revision note (ROAD-2):** Validation criterion 8 was added to test the `stream.button()` → command → Promise pattern early. If this test reveals limitations, the fallback (text-based approval) can be designed into the workflow engine's step definition schema in M3 rather than retrofitted in M9.
> 

**Dependencies:** None (first milestone).

**Complexity:** S (Small) — standard scaffolding, well-documented VS Code APIs.

---

## Milestone 1: Mock Infrastructure + Model Resolver — ✅ COMPLETE

**What is built:** The mock/stub layer for the Language Model API and the model resolver that maps tiers to available models. This is permanent test infrastructure, not throwaway scaffolding.

**Specific modules:**

- `spawner/agent-spawner.ts` — shell with injectable model interface
- `engine/model-resolver.ts` — maps model tiers (free/standard/premium) to available models via `vscode.lm.selectChatModels()`
- `test/mocks/mock-language-model.ts` — mock implementation of the Language Model API that returns canned responses
- `test/mocks/mock-chat-response-stream.ts` — mock `vscode.ChatResponseStream`
- `test/fixtures/` — canned agent responses for each workflow step type

**Key design decisions:**

- `AgentSpawner` takes a model provider interface (production: VS Code LM API; test: mock)
- Mock supports configurable behavior: success, failure, timeout, partial response
- Mock tracks call count and arguments for assertion in tests
- Model resolver returns models sorted by preference within each tier

**What works at the end:**

- Unit tests can spawn mock agents and verify prompt construction, tool scoping, and result handling
- Model resolver correctly maps tiers to models (tested with mock model catalog)
- Escalation logic can be tested: mock first call to fail, second to succeed with different tier

**Validation criteria:**

1. Mock language model accepts prompts and returns configurable canned responses
2. Mock tracks all calls (prompt, model, tools) for test assertions
3. Model resolver maps `free` → GPT-4.1/GPT-5 mini, `standard` → Claude Sonnet/GPT-5.2, `premium` → Claude Opus
4. Model resolver falls back to lower tier when requested tier has no available models
5. Mock can simulate failure (throws/rejects) for testing escalation
6. All tests pass with `npm run test`

**Dependencies:** Milestone 0 (project exists, types defined).

**Complexity:** S — well-bounded interfaces, no external dependencies.

---

## Milestone 2: Intent Classifier — ✅ COMPLETE

**What is built:** The two-tier intent classification system. Tier 1 is local keyword/regex matching. Tier 2 is LLM-based classification (tested with mocks, manually verified with real models).

**Specific modules:**

- `classifier/intent-classifier.ts` — two-tier classification logic
- `classifier/intent-patterns.ts` — pattern map: regex/keyword patterns → intent types with weights
- `classifier/intent-classifier.test.ts` — 100+ prompt examples with expected classifications

**Specification for AI agent:**

The intent classifier receives a developer chat prompt (string) and returns a `ClassificationResult` with intent type, confidence score (0.0–1.0), matched signals, and a flag indicating whether LLM classification is needed. The local classifier computes confidence from weighted signal matches: primary signals carry weight 0.4, secondary signals 0.2. Negative signals (e.g., "don't fix") reduce confidence. If multiple intents score above 0.3, the highest wins but confidence is capped at 0.6 (ambiguous). If local confidence is below 0.7, the `requiresLLM` flag is set to true. The LLM classifier (Tier 2) appends a structured-output request to the system prompt, asking the model to return JSON with intent and reasoning before the main response. This is a double-duty call — one LLM invocation serves both classification and response.

**Validation criteria:**

1. Local classifier achieves ≥90% accuracy on a test suite of 100+ prompts spanning all 8 intent types
2. Ambiguous prompts (matching multiple intents) produce confidence ≤0.6 and set `requiresLLM: true`
3. Negative signals ("don't fix this, just explain") correctly reduce intent confidence
4. LLM classifier (via mock) correctly parses `{"intent": "bug_fix", "reasoning": "..."}` from model output
5. Fallback to `general_chat` when no signals match
6. Classification latency <10ms for local tier (no I/O)
7. All 100+ test cases pass

**Dependencies:** Milestone 0 (types), Milestone 1 (mock layer for LLM tier).

**Complexity:** S — pattern matching, no external dependencies, well-defined test suite.

---

## Milestone 3: Workflow Engine Core — ✅ COMPLETE

**What is built:** The state machine workflow engine — sequential step execution, retry with escalation, cancellation, and timeout handling. No parallel execution yet (added in Milestone 9).

**Specific modules:**

- `engine/workflow-engine.ts` — core execution loop, state transitions
- `engine/step-executor.ts` — executes individual steps, manages retry/escalation
- `engine/workflow-engine.test.ts` — state transition tests
- `engine/step-executor.test.ts` — retry/escalation tests
- Updated `types.ts` — `WorkflowDefinition`, `WorkflowStep`, `WorkflowContext`, `WorkflowResult`, execution states enum

**Specification for AI agent:**

The workflow engine is a finite state machine. It receives a `WorkflowDefinition` (list of steps with handlers) and a `WorkflowContext` (prompt, intent, project model reference, chat response stream, cancellation token). It traverses steps sequentially. Each step is executed by the step executor, which spawns a subagent via the Agent Spawner. On step failure, the executor retries: (1) same model with refined prompt including error context, (2) next higher model tier, (3) after 3 failures, report to developer and pause. The engine respects cancellation tokens at every step boundary. Workflow states are: PENDING, RUNNING, RETRYING, PAUSED, COMPLETED, FAILED, CANCELLED. The engine streams progress updates to the chat response stream at each state transition.

> **Note (ROAD-3):** Escalation context enrichment (injecting test failure output back into step 3's prompt) is handled by the step executor's retry logic, not by the `WorkflowStep.condition` function. The `condition` function only selects the next step ID for branching; it does not modify the workflow context.
> 

**Validation criteria:**

1. A 4-step sequential workflow completes successfully with mock agents
2. Step results are passed to subsequent steps as context
3. A step that fails on attempt 1 succeeds on attempt 2 (same tier, refined prompt)
4. A step that fails on free tier succeeds on standard tier (escalation)
5. A step that fails 3 times produces a PAUSED state with error summary
6. Cancellation during step 2 of 4 produces CANCELLED state; steps 3–4 are not executed
7. Step timeout (e.g., 100ms in test) triggers failure and retry
8. Progress updates are streamed to the mock chat response stream

**Dependencies:** Milestone 1 (mock infrastructure, model resolver), Milestone 0 (types).

**Complexity:** M (Medium) — state machine with non-trivial transition logic and concurrency considerations.

---

## Milestone 4: Agent Spawner + Prompt Builder — ✅ COMPLETE

**What is built:** The agent spawner that creates ephemeral subagents with role-specific prompts, scoped tools, and model selection. The prompt builder that constructs three-layer prompts (role + context + task).

**Specific modules:**

- `spawner/agent-spawner.ts` — full implementation (subagent creation, parallel execution support)
- `spawner/prompt-builder.ts` — three-layer prompt construction
- `spawner/tool-registry.ts` — global tool registry with scoping per step type
- `spawner/agent-spawner.test.ts`
- `spawner/prompt-builder.test.ts`

**Specification for AI agent:**

The agent spawner receives an `AgentConfig` (role, model tier, tools, prompt template, context, timeout) and returns an `AgentResult` (text output, tool call results, token usage, status). It uses the model resolver to select a model for the given tier, constructs the prompt via the prompt builder, scopes the tool set from the tool registry, and sends the request via the VS Code Language Model API (`vscode.lm.sendChatRequest()`). The prompt builder constructs prompts from three layers: (1) role prompt defining the agent's identity and constraints, (2) context injection with serialized project model data (only relevant context to minimize tokens), (3) task prompt with variable substitution from workflow context. The tool registry maintains all available VS Code LM tools and returns scoped subsets per step type: research steps get read-only tools, implementation steps get read/write tools, review steps get read + test tools.

**Validation criteria:**

1. Spawned agent receives correct three-layer prompt (verified by mock inspection)
2. Research step agent has read-only tools; implementation step agent has read/write tools
3. `spawnParallel` with 3 agents: if one fails, others still complete (Promise.allSettled)
4. Token usage is tracked per agent call
5. Timeout is enforced (mock simulates slow response)
6. Agent result includes structured output from mock

**Dependencies:** Milestone 1 (mock infrastructure, model resolver), Milestone 0 (types).

**Complexity:** M — prompt construction and tool scoping require careful design.

---

## Milestone 5: Lazy Project Model (Node.js/TypeScript) — ✅ COMPLETE

**What is built:** The project analyzer (Node.js plugin only) and in-memory project model. Reads `package.json`, lock files, `tsconfig.json`, test runner config, linter/formatter config. Stores results in a SQLite database. Provides serialized context for LLM prompts.

**Specific modules:**

- `analyzer/project-analyzer.ts` — orchestrates analysis, generic plugin interface
- `analyzer/dependency-scanner.ts` — reads `package.json`, detects lock file type
- `analyzer/directory-scanner.ts` — `fast-glob` directory scanning, role assignment
- `model/project-model.ts` — in-memory model with debounced SQLite writes
- `model/database.ts` — SQLite operations via `better-sqlite3`
- `analyzer/project-analyzer.test.ts`
- `model/project-model.test.ts`
- `test/fixtures/` — minimal Node.js test projects (with `package.json`, `tsconfig.json`, etc.)

**Specification for AI agent:**

The project analyzer has a generic interface (`AnalyzerPlugin`) from the start, but only the Node.js implementation exists. The Node.js analyzer reads: `package.json` (dependencies, devDependencies, scripts, engines), lock files (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`), `tsconfig.json`, test runner config (`vitest.config.*`, `jest.config.*`), linter config (`.eslintrc.*`), formatter config (`.prettierrc.*`). Results populate the in-memory `ProjectModel`, which is backed by SQLite at `.github/.roadie/project-model.db`. The model provides a `toContext()` method that serializes the tech stack, directory structure, and detected commands into a string suitable for LLM prompt injection. The database is created lazily on first analysis. Debounced writes batch every 5 seconds maximum.

**Validation criteria:**

1. Analyzing a Next.js + Prisma + Vitest project detects: TypeScript, Next.js (version), Prisma, Vitest, pnpm (from pnpm-lock.yaml), ESLint, Prettier
2. Analyzing a bare Express.js + Jest project detects: JavaScript, Express, Jest, npm
3. `toContext()` output contains tech stack, directory structure, and commands in a readable format
4. SQLite database is created at `.github/.roadie/project-model.db` on first analysis
5. Model loads from SQLite on extension re-activation (verified by activating twice)
6. Stale detection triggers re-analysis when `package.json` modification time > last analyzed time
7. Directory scanner ignores `node_modules`, `.git`, `dist`, `build`, `coverage`
8. All tests pass against fixture projects in `test/fixtures/`

**Dependencies:** Milestone 0 (project structure, types).

**Complexity:** M — multiple file parsers, SQLite integration, debounced writes.

---

## Milestone 5.5: Wire-Up Spike (End-to-End Validation) — ✅ COMPLETE

> **Revision note (ROAD-1):** This milestone was added to validate the full data flow before investing in the complete 8-step bug-fix workflow in M6. Five milestones of infrastructure (M1–M5) precede the first end-to-end test. If the Chat Participant → Intent Classifier → Workflow Engine → Agent Spawner → Project Model data flow has a design flaw, this spike catches it early.
> 

**What is built:** A minimal integration test that validates the complete active-mode data flow with a trivial 1-step workflow.

**What works at the end:**

1. The Chat Participant receives a hardcoded prompt
2. The intent classifier returns a hardcoded intent
3. The workflow engine executes a 1-step workflow (not the full 8-step bug fix)
4. The agent spawner sends one mock LLM call
5. The result streams back through the Chat Participant

**Validation criteria:**

1. A prompt sent to `@roadie` flows through all components and produces a streamed response
2. The project model context is injected into the agent's prompt
3. The full data flow completes in under 5 seconds (with mock LLM)
4. If any component boundary is broken, the test fails with a clear error identifying the boundary

**Dependencies:** Milestones 1–5.

**Complexity:** S — integration wiring only, no new logic.

---

## Milestone 6: Bug Fix Workflow (End-to-End) — ✅ COMPLETE

**What is built:** The first complete workflow — bug fix — wired end-to-end: chat input → intent classification → workflow engine → agent spawner → project model context → streamed response. This is the "first proof" milestone.

**Specific modules:**

- `engine/definitions/bug-fix.ts` — declarative bug-fix workflow definition (8 steps)
- Updated `shell/chat-participant.ts` — routes classified intents to workflow engine
- Updated `spawner/prompt-builder.ts` — bug-fix-specific prompt templates (Diagnostician, Fixer roles)
- Integration test: end-to-end chat → classification → workflow → response

**Specification for AI agent:**

Wire the full active-mode data flow. The Chat Participant handler receives a prompt, passes it to the intent classifier. If the intent is `bug_fix`, it creates a `WorkflowContext` (with project model, chat response stream, cancellation token) and calls `workflowEngine.execute('bug_fix', context)`. The bug-fix workflow definition has 8 sequential steps as specified in the PDD: (1) locate error source, (2) diagnose root cause, (3) generate and apply fix, (4) verify fix (run test suite), (5) scan for sibling bugs, (6) fix siblings, (7) add regression guard, (8) generate summary. Each step has a prompt template, agent role, tool scope, and model tier. Step 4 uses shell execution to run the project's test command (detected from `package.json` scripts). Escalation: if step 4 fails, return to step 3 with test output as additional context and escalate to standard tier. This milestone assumes the project has a working test command.

**Validation criteria:**

1. End-to-end integration test: mock agents process all 8 steps and produce a structured result
2. Chat participant correctly routes `bug_fix` intent to the workflow engine
3. Each step's prompt includes project model context (tech stack, directory structure)
4. Step 4 (verify fix) invokes shell command with test runner from project model
5. Escalation: when step 4 fails, step 3 retries with test output in prompt and standard-tier model
6. Three consecutive failures in step 3 produce a PAUSED state with diagnostic summary
7. Cancellation at any step stops execution and reports partial results
8. Manual test: run in Extension Development Host against a real Node.js project with a planted bug

**Dependencies:** Milestones 1–5, Milestone 5.5 (wire-up spike validates data flow first).

**Complexity:** L (Large) — first full integration of all components, complex wiring.

---

## Milestone 7: Basic File Generation — ✅ COMPLETE

**What is built:** The file generator that produces `.github/copilot-instructions.md` and `AGENTS.md` from the project model. Section ownership markers. No edit detection yet (added in Phase 1.5).

**Specific modules:**

- `generator/file-generator.ts` — orchestrates file generation
- `generator/section-manager.ts` — section ownership markers, hash tracking
- `generator/templates/copilot-instructions.ts` — template for `copilot-instructions.md`
- `generator/templates/agent-definitions.ts` — template for `AGENTS.md`
- `generator/file-generator.test.ts`
- `generator/section-manager.test.ts`

**Validation criteria:**

1. `copilot-instructions.md` contains accurate tech stack from project model
2. `AGENTS.md` contains project overview and cross-tool instructions
3. Files contain `<!-- roadie:start:... -->` and `<!-- roadie:end:... -->` markers
4. SHA-256 hashes are stored in SQLite after generation
5. Regeneration with unchanged model produces no file write (hash comparison)
6. `.github/.roadie/.gitignore` exists and contains `project-model.db`
7. Generated files appear as unstaged changes in VS Code Source Control

**Dependencies:** Milestone 5 (project model), Milestone 6 (workflow trigger for generation).

**Complexity:** S — template strings, hash comparison, file I/O.

---

## Milestone 8: Passthrough Mode + General Chat Enhancement — ✅ COMPLETE

**What is built:** When the intent classifier returns `general_chat`, Roadie enriches the prompt with project model context and forwards to the underlying model.

**Validation criteria:**

1. General question is correctly classified as `general_chat` (no workflow triggered)
2. Prompt includes serialized project context from `toContext()`
3. Response (via mock) receives the enriched prompt
4. Latency overhead of context injection is <100ms

**Dependencies:** Milestone 2 (intent classifier), Milestone 5 (project model).

**Complexity:** S — minimal wiring, straightforward context injection.

---

## Milestone 9: Feature Development Workflow — ✅ COMPLETE

**What is built:** The feature development workflow with plan approval (human-in-the-loop) and parallel layer delegation.

**Specific modules:**

- `engine/definitions/feature.ts` — declarative feature workflow definition (7 steps)
- Updated `engine/workflow-engine.ts` — WAITING_PARALLEL state, `Promise.allSettled()` for parallel branches
- Updated `spawner/prompt-builder.ts` — Planner, Database Agent, Backend Agent, Frontend Agent prompt templates

**Validation criteria:**

1. Plan approval buttons render in chat and pause the workflow
2. "Revise Plan" loops back to step 1 with feedback
3. Parallel execution: 3 mock agents run concurrently via `Promise.allSettled()`
4. One parallel branch failing does not block others
5. Integration step receives results from all completed branches
6. End-to-end test with mock agents completes all 7 steps

**Dependencies:** Milestone 3 (workflow engine), Milestone 4 (agent spawner), Milestone 5 (project model).

**Complexity:** L — parallel execution, human-in-the-loop interaction, most complex workflow.

---

## Milestone 10: Refactoring Workflow — ✅ COMPLETE

**What is built:** The refactoring workflow with characterization test writing, incremental refactoring, and safety verification after each step.

**Validation criteria:**

1. Characterization tests are generated before any refactoring begins
2. Each incremental refactoring step is followed by test execution
3. Failed test after a refactoring step triggers revert and alternative approach
4. Public API invariant: workflow detects and stops if a refactoring would change public interfaces
5. Before/after summary includes diff analysis

**Dependencies:** Milestone 3 (workflow engine), Milestone 4 (agent spawner).

**Complexity:** M — inner loop logic requires careful state management.

---

## Milestone 11: Code Review Workflow — ✅ COMPLETE

**What is built:** The 5-pass parallel code review workflow covering security, performance, code quality, test coverage, and project standards.

**Validation criteria:**

1. All 5 review passes run concurrently via `Promise.allSettled()`
2. Results are consolidated with findings categorized by severity
3. Security pass uses Tier 1 model; others use Tier 0
4. Review targets git diff (verified by mock checking shell command for `git diff`)
5. Empty diff produces "No changes to review" response

**Dependencies:** Milestone 3 (workflow engine with parallel support from M9), Milestone 4 (agent spawner).

**Complexity:** M — parallel passes are structurally similar to feature workflow's parallel delegation.

---

## Milestone 12: No-Tests Fallback + Remaining Workflows — ✅ COMPLETE

**What is built:** The "no-tests" fallback for bug fix and refactoring workflows, plus the documentation, dependency management, and onboarding workflows.

**Validation criteria:**

1. Bug fix on a project without `test` script: workflow completes with verification note, generates test file
2. Documentation workflow produces docs from code analysis, not existing docs
3. Dependency workflow executes upgrades one at a time, reverts on test failure
4. Onboarding workflow uses project model to generate architecture overview
5. All workflow definitions have corresponding tests with mock agents

**Dependencies:** Milestones 3–5, Milestone 6 (for no-tests fallback patterns).

**Complexity:** M — multiple workflows, but each follows established patterns.

---

## Milestone 13: Configuration Model + Polish — ✅ COMPLETE

**What is built:** The configuration model (all settings in `.vscode/settings.json` under the `roadie` namespace), auto-commit option, test timeout configuration, model preference override, `roadie.testCommand` override. Plus: marketplace metadata, icon, README, changelog.

**Validation criteria:**

1. Changing `roadie.testTimeout` to 10 causes test step to timeout after 10 seconds
2. Setting `roadie.modelPreference` to "quality" starts workflows at Tier 1
3. Setting `roadie.autoCommit` to true creates git commits for generated files
4. `Roadie: Reset` deletes `.github/.roadie/project-model.db` and reinitializes
5. `Roadie: Rescan Project` triggers full project model rebuild
6. Extension validates and packages with `vsce package`

**Dependencies:** All prior milestones.

**Complexity:** S — configuration wiring, marketplace metadata.

---

## Milestone 14: Marketplace Publishing (Phase 1 Complete) — ⏭ SKIPPED (private tool, not yet published)

**What is built:** First marketplace publish as Preview/Pre-release. Final testing, packaging, and submission.

**Validation criteria:**

1. `vsce package` produces a valid `.vsix` file
2. `.vsix` installs cleanly in a fresh VS Code instance
3. All 7 workflows run end-to-end in Extension Development Host
4. README is accurate and includes screenshots
5. Published as Preview/Pre-release on marketplace

**Dependencies:** All Phase 1 milestones.

**Complexity:** S — packaging and publishing.

---

# Phase 1.5 Milestones — Passive Mode

## Milestone 15: File Watcher + Incremental Model Updates — ✅ COMPLETE

**What is built:** VS Code `FileSystemWatcher` monitoring dependency files and config files. When watched files change, the project model updates incrementally.

**Watch patterns:** `**/package.json`, `**/tsconfig.json`, `**/.eslintrc*`, `**/.prettierrc*`, `**/vitest.config*`, `**/jest.config*`

**Validation criteria:**

1. Adding a new dependency to `package.json` and saving triggers model update
2. Debounce: saving `package.json` twice within 500ms triggers only one scan
3. Model update is incremental (only re-scans the changed file's category)
4. Status bar reflects scanning state
5. File watcher respects workspace trust (disabled in untrusted workspaces)

**Dependencies:** Phase 1 complete (Milestone 14).

**Complexity:** S — VS Code FileSystemWatcher is well-documented, debounce is standard.

---

## Milestone 16: Automatic File Regeneration — ✅ COMPLETE

**What is built:** When the project model changes, automatically regenerate affected `.github/` files. Respects section ownership markers.

**Validation criteria:**

1. Adding a dependency to `package.json` → `copilot-instructions.md` updates within 2 seconds
2. Developer edits content outside Roadie markers → edits preserved on regeneration
3. Developer edits content inside Roadie markers → merge append with `<!-- roadie:merged:timestamp -->`
4. Developer removes Roadie markers → file becomes fully human-owned, Roadie logs warning and skips
5. File open and unsaved in editor → write deferred until save
6. Identical content → no file write

**Dependencies:** Milestone 15 (file watcher), Milestone 7 (file generator).

**Complexity:** M — merge logic for human-edited sections requires careful handling.

---

## Milestone 17: Pattern Detection (Phase 1.5 Deep Model) — ✅ COMPLETE

**What is built:** Regex-based pattern detection for coding patterns, git conventions, and import ordering.

**Detected patterns:** Export style, test convention, error handling, import ordering, commit convention, async patterns.

**Validation criteria:**

1. Project using barrel exports → pattern detected with ≥0.8 confidence
2. Project using conventional commits → pattern detected from git log
3. Sampling cap: max 100 files per pattern scan, completes in <500ms per pattern
4. Detected patterns appear in `copilot-instructions.md` regeneration
5. Low-confidence patterns (<0.5) are stored but not included in generated files

**Dependencies:** Milestone 15 (file watcher for triggering), Milestone 5 (project model).

**Complexity:** M — regex pattern matching across sampled files, confidence scoring.

---

## Milestone 18: Learning Database + Edit Tracking — ✅ COMPLETE

**What is built:** The learning database with file snapshot storage, workflow history recording, and edit tracking (opt-in). Pruning policy enforcement.

**Validation criteria:**

1. After file generation, a snapshot exists in the database with source "roadie"
2. After developer edit + regeneration, a snapshot exists with source "human"
3. Workflow history records type, status, duration for each workflow execution
4. Pruning removes snapshots beyond 50 per file
5. Pruning removes workflow history beyond 100 entries
6. Database size stays under 10 MB after 200 workflow executions on a typical project
7. Edit tracking and workflow history are disabled by default (opt-in only)

**Dependencies:** Milestone 7 (file generator), Milestone 3 (workflow engine).

**Complexity:** M — SQLite schema additions, pruning logic, opt-in configuration.

---

## Milestone 19: Extended File Generation — ✅ COMPLETE

**What is built:** Full file generation catalog: per-language instructions, agent definitions, skill definitions, hooks, PR template, issue templates.

**Validation criteria:**

1. TypeScript project → `typescript.instructions.md` generated with TS-specific conventions
2. React project → `react.instructions.md` generated with React patterns
3. Running bug-fix workflow → `debugger.agent.md` generated
4. ESLint detected → `pre-commit.json` hook generated
5. All generated files have Roadie section markers
6. Total generated files ≤20

**Dependencies:** Milestone 7 (file generator), Milestone 17 (pattern detection for content).

**Complexity:** M — many templates, but each follows the same pattern.

---

## Milestone 20: Sidebar View + Status Dashboard — ⏭ DEFERRED (sidebar not implemented; status available via Output channel and status bar)

**What is built:** Sidebar webview showing project model status, detected patterns, generated files, and workflow history.

**Validation criteria:**

1. Sidebar displays detected tech stack accurately
2. Sidebar updates when `package.json` changes trigger model update
3. Sidebar shows recent workflow history with status indicators
4. Sidebar renders without errors in Extension Development Host

**Dependencies:** Milestone 18 (learning database for history), Milestone 17 (patterns).

**Complexity:** S — standard webview implementation, read-only dashboard.

---

# Phase 2 Milestones — MCP Server

## Milestone 21: MCP Server Scaffold — ⏳ NOT YET STARTED (Phase 2)

**What is built:** MCP server skeleton using `@modelcontextprotocol/sdk`, stdio-based transport, spawned by the extension.

**Validation criteria:**

1. MCP server process starts and responds to `initialize`
2. `ping` returns a valid response
3. Server exits cleanly when extension deactivates
4. Server restart on crash (process exits unexpectedly)

**Dependencies:** Phase 1.5 complete.

**Complexity:** S — MCP SDK handles transport, minimal boilerplate.

---

## Milestone 22: MCP Tools — Project Info + Context — ⏳ NOT YET STARTED (Phase 2)

**What is built:** MCP tools wrapping `ProjectAnalyzer.analyze()` and `ProjectModel.toContext()`. External tools can query Roadie's project model.

**Validation criteria:**

1. MCP tool `roadie/project-info` returns structured JSON with tech stack, dependencies
2. MCP tool `roadie/project-context` returns serialized context string
3. Tools respond within 1 second
4. Tools return graceful errors when project model is not yet built

**Dependencies:** Milestone 21 (MCP server), Milestone 5 (project model).

**Complexity:** S — thin wrappers around existing interfaces.

---

## Milestone 23: MCP Tools — Workflow Execution + File Generation — ⏳ NOT YET STARTED (Phase 2)

**What is built:** MCP tools wrapping `WorkflowEngine.execute()` and `FileGenerator.generate()`. External tools can trigger Roadie workflows and file generation.

**Validation criteria:**

1. MCP tool `roadie/workflow-execute` triggers a bug-fix workflow and returns results
2. MCP tool `roadie/file-generate` regenerates `copilot-instructions.md` and returns the content
3. MCP tool `roadie/learning-data` returns recent workflow history
4. Long-running workflows stream progress via MCP notifications

**Dependencies:** Milestone 22, all Phase 1 workflows.

**Complexity:** M — workflow execution via MCP requires progress streaming over stdio.

---

# Module Build Order

The following table shows the build order derived from the dependency graph.

| Order | Module | Milestone | Depends On |
| --- | --- | --- | --- |
| 1 | `types.ts` | M0 | — |
| 2 | `extension.ts` (shell) | M0 | types |
| 3 | `chat-participant.ts` (shell) | M0 | types |
| 4 | `status-bar.ts` | M0 | types |
| 5 | `container.ts` | M0 | types |
| 6 | Mock infrastructure | M1 | types |
| 7 | `model-resolver.ts` | M1 | types, mocks |
| 8 | `intent-classifier.ts` | M2 | types, mocks |
| 9 | `intent-patterns.ts` | M2 | types |
| 10 | `workflow-engine.ts` | M3 | types, mocks, model-resolver |
| 11 | `step-executor.ts` | M3 | types, model-resolver |
| 12 | `agent-spawner.ts` | M4 | types, model-resolver |
| 13 | `prompt-builder.ts` | M4 | types |
| 14 | `tool-registry.ts` | M4 | types |
| 15 | `database.ts` | M5 | types |
| 16 | `project-model.ts` | M5 | types, database |
| 17 | `project-analyzer.ts` | M5 | types, project-model |
| 18 | `dependency-scanner.ts` | M5 | types |
| 19 | `directory-scanner.ts` | M5 | types |
| 19.5 | Wire-up spike | M5.5 | all M1–M5 modules |
| 20 | `bug-fix.ts` (workflow def) | M6 | all engine + spawner + model |
| 21 | `file-generator.ts` | M7 | types, project-model, database |
| 22 | `section-manager.ts` | M7 | types, database |
| 23 | `copilot-instructions.ts` (template) | M7 | types, project-model |
| 24 | `agent-definitions.ts` (template) | M7 | types, project-model |
| 25 | `feature.ts` (workflow def) | M9 | engine, spawner |
| 26 | `refactor.ts` (workflow def) | M10 | engine, spawner |
| 27 | `review.ts` (workflow def) | M11 | engine, spawner |
| 28 | `document.ts` (workflow def) | M12 | engine, spawner |
| 29 | `dependency.ts` (workflow def) | M12 | engine, spawner |
| 30 | `onboard.ts` (workflow def) | M12 | engine, spawner |
| 31 | `pattern-detector.ts` | M17 | types, project-model |
| 32 | `learning-database.ts` | M18 | types, database |
| 33 | All remaining templates | M19 | file-generator, project-model |
| 34 | `sidebar-provider.ts` | M20 | project-model, learning-database | *(deferred — not yet implemented)* |
| 35 | `mcp-server.ts` | M21 | extension |
| 36 | MCP tools | M22–23 | mcp-server, all core interfaces |

---

# Module Specifications (AI Agent Build Prompts)

**types.ts:** Define all shared TypeScript interfaces and types for the Roadie codebase. This file is the single source of truth for all module boundary contracts. Include: `ClassificationResult` and `IntentType` (8 intent types), `WorkflowDefinition`/`WorkflowStep`/`WorkflowContext`/`WorkflowResult` (workflow engine), execution state enum (PENDING, RUNNING, WAITING_PARALLEL, RETRYING, PAUSED, COMPLETED, FAILED, CANCELLED), `AgentConfig`/`AgentResult`/`AgentRole` (agent spawner), `ProjectModel` interface and `TechStackEntry`/`DirectoryNode`/`DetectedPattern`/`DeveloperPreferences`, `GeneratedFile`/`GeneratedFileType` (file generator), `FileSnapshot`/`WorkflowHistoryEntry` (learning database), and all enum types. Every interface must have JSDoc comments explaining its purpose and usage.

**intent-classifier.ts:** Implement two-tier intent classification. Tier 1: local keyword/regex matching using the pattern map from `intent-patterns.ts`. Compute confidence from weighted signal matches (primary 0.4, secondary 0.2). Cap confidence at 0.6 when multiple intents score above 0.3. Set `requiresLLM: true` when confidence < 0.7. Tier 2: append a structured-output JSON request to the system prompt asking the model to classify before responding. Parse the JSON classification from the model's response. Fallback to `general_chat` on any error. Export `classify(prompt)`, `parseClassification(responseText)`, and `getClassificationPromptPrefix()`.

**workflow-engine.ts:** Implement a finite state machine that executes multi-step workflows. Accept a `WorkflowDefinition` and `WorkflowContext`. Traverse steps sequentially (Phase 1) and in parallel (via `Promise.allSettled()` for steps with `type: 'parallel'`). At each step, call the step executor. Manage state transitions: PENDING → RUNNING → (RETRYING | WAITING_PARALLEL) → COMPLETED | PAUSED | FAILED | CANCELLED. Stream progress updates to the chat response stream at each transition. Respect `CancellationToken` at every step boundary. Support conditional steps via `condition` predicates. Export `execute()`, `cancel()`, `getActiveWorkflows()`, `registerWorkflow()`.

**step-executor.ts:** Execute individual workflow steps. Receive a step definition and workflow context. Use the agent spawner to create a subagent. Manage retry/escalation: attempt 1 uses the specified tier with the original prompt; attempt 2 uses the same tier with refined prompt including error context; attempt 3 escalates to the next tier. After 3 failures, return a PAUSED status with error summary. Enforce step timeout via `Promise.race()` with a timer. Return `StepResult` with output, status, attempts count, and model used.

**project-analyzer.ts:** Orchestrate project analysis with a plugin architecture. Define `AnalyzerPlugin` interface with `canAnalyze(workspaceRoot): boolean` and `analyze(workspaceRoot, scope): Promise<AnalysisResult>`. Implement `NodeJsAnalyzerPlugin` that reads `package.json`, lock files, `tsconfig.json`, test runner config, linter/formatter config. Store results in the project model. Support scoped analysis (`full`, `dependencies`, `patterns`, `structure`). Support `invalidate(paths)` for incremental updates. All detection is regex/string-based, never AST parsing.

**project-model.ts:** In-memory representation of project state. Load from SQLite at activation. Expose typed accessors (`getTechStack()`, `getDirectoryStructure()`, `getPatterns()`, etc.). Implement `toContext()` that serializes the model into a structured text block for LLM prompt injection (tech stack, directory structure, commands, patterns). Accept optional `maxTokens` parameter for token budgeting. Implement debounced writes to SQLite (batch every 5 seconds max). Track dirty state. Support incremental `update(delta)` for partial model changes.

**database.ts:** SQLite operations via `better-sqlite3`. Create tables on first access (schema from TAD Section 3.1). Implement migrations via `schema_version` table. CRUD for tech_stack, directory_structure, file_snapshots, workflow_history, detected_patterns, developer_preferences. Implement `PRAGMA integrity_check` on open — if corrupt, delete and recreate. Located at `.github/.roadie/project-model.db`.

**file-generator.ts:** Orchestrate generation of all `.github/` files. Use the section manager for ownership tracking. Use template functions (one per file type) that accept the project model and return string content. Before writing, hash-compare new content against existing file — skip if identical. Handle deferred writes for files open in the editor. Generate `.github/.roadie/.gitignore` with `project-model.db`. Export `generate(fileType)`, `generateAll()`, `detectHumanEdits(filePath)`, `merge(generated, existing)`.

**section-manager.ts:** Manage section ownership in generated Markdown files. Parse `<!-- roadie:start:section-name -->` and `<!-- roadie:end:section-name -->` markers. Compute SHA-256 hashes of section content. Store hashes in the database. Detect human edits by comparing current file section hash against stored hash. Implement merge strategy: no edits → replace; human edits → append below existing with `<!-- roadie:merged:timestamp -->`; markers removed → file is human-owned, skip.

**learning-database.ts:** SQLite storage for file snapshots, workflow outcomes, and detected patterns. Implement `recordSnapshot()`, `getSnapshots()`, `recordWorkflowOutcome()`, `getWorkflowHistory()`, `recordPattern()`, `prune()`. Retention: last 50 snapshots per file, last 100 workflow entries. Pruning runs on extension activation. Share the same SQLite database as the project model.

---

# Core Interface Definitions

See the companion TAD (Technical Architecture Document) for the complete TypeScript interface definitions. The canonical interfaces are maintained there to avoid duplication. Key interfaces: `ClassificationResult`, `IntentType`, `WorkflowDefinition`, `WorkflowStep`, `WorkflowContext`, `WorkflowResult`, `StepResult`, `AgentConfig`, `AgentResult`, `AgentRole`, `ModelTier`, `ToolScope`, `TechStackEntry`, `DirectoryNode`, `DetectedPattern`, `ProjectContext`, `ProjectCommand`, `GeneratedFileType`, `GeneratedFile`, `SectionOwnership`, `FileSnapshot`, `WorkflowHistoryEntry`, `AnalyzerPlugin`, `AnalysisScope`, `AnalysisResult`, `RoadieConfig`.

> **Note:** The `DetectedPattern` interface has `confidence` at the top level (aggregate confidence used by file generators and workflow engine) and inside the `evidence` object (confidence based on the specific evidence sample). These serve different purposes and are both retained.
> 

---

# Risk Register

## Technical Risks

| # | Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- | --- |
| R1 | VS Code Language Model API changes or is unstable | Medium | High | Abstract all LM API calls behind the model resolver and agent spawner interfaces. If the API changes, only two modules need updating. Pin minimum VS Code version in `engines` field. |
| R2 | `stream.button()` for plan approval doesn't work as expected or has limitations | Medium | Medium | Tested in Milestone 0 (validation criterion 8). If buttons don't work, fall back to text-based approval ("Type APPROVE or REVISE"). This is a graceful degradation, not a blocker. |
| R3 | Intent classification accuracy is too low for reliable workflow routing | Medium | High | Two-tier design is the mitigation. If local classifier is unreliable, the LLM fallback catches it. If LLM classification is unreliable, add a confirmation step. Track misclassifications via opt-in workflow history. |
| R4 | Test runner invocation fails on diverse project configurations | High | Medium | Start with the simplest case (`npm test` / `npx vitest`). Detect the test command from `package.json` scripts. If no test command is found, use the no-tests fallback. Never assume a specific test runner — always detect. Add a `roadie.testCommand` override setting for edge cases. |
| R5 | Shell command execution is blocked by VS Code workspace trust | Low | Medium | Respect workspace trust from day one. In untrusted workspaces, workflows that require shell execution report this limitation and offer a reduced-functionality mode. |
| R6 | SQLite database corruption from crashes during writes | Low | High | Debounced writes (5s batches) minimize write frequency. `PRAGMA integrity_check` on activation with delete-and-rebuild recovery. WAL mode for crash resilience. Database is rebuiltable from source files. |
| R7 | Premium model quota exhaustion (300 requests/month for Copilot Pro) | Medium | Medium | Cost-awareness is structural. All workflows start at Tier 0 (free). Tier 2 (premium) is last-resort escalation. Premium usage tracking requires a model-to-multiplier mapping maintained in the model resolver. Since GitHub may change multipliers, tracking tier usage (% of calls at each tier) is sufficient for cost awareness without needing exact multiplier values. |
| R8 | Generated file section merging produces confusing results when developer edits extensively | Medium | Low | The merge strategy (append below, not overwrite) is conservative. The key invariant is "never lose human work." |
| R9 | File watcher fires too frequently and causes performance issues | Low | Medium | Debounce at 500ms. Watch only specific known file patterns, not broad globs. Ignore `node_modules`, `.git`, and build output directories. Cap at 500 watched paths. VS Code's built-in FileSystemWatcher handles the heavy lifting. |
| R10 | Workflow execution is too slow (multiple sequential LLM calls) | Medium | High | Tier 0 models (GPT-5 mini, GPT-4.1) are fast. Parallel execution where possible. Progress streaming keeps the developer informed. Timeout enforcement prevents runaway steps. |

## Decision Points (Hard to Reverse)

| # | Decision | When | Options | Recommendation |
| --- | --- | --- | --- | --- |
| D1 | Chat Participant vs. invisible interception | Before M0 | (a) `@roadie` dropdown selection, (b) intercept all Copilot chat silently | Already decided: `@roadie` dropdown. Explicit selection respects developer agency. |
| D2 | SQLite vs. JSON file storage for project model | Before M5 | (a) SQLite via `better-sqlite3`, (b) JSON files | SQLite. Supports structured queries, handles concurrent reads/writes, has integrity checking. |
| D3 | Template strings vs. template engine for file generation | Before M7 | (a) TypeScript template strings, (b) Handlebars/Mustache/EJS | Template strings. Full TypeScript type safety, no additional dependency, simple for AI agents to modify. |
| D4 | VS Code FileSystemWatcher vs. chokidar | Before M15 | (a) Built-in VS Code watcher, (b) chokidar | Already decided: VS Code built-in. Integrates with workspace trust, works in remote dev, zero dependencies. |
| D5 | Single ecosystem (Node.js) vs. multi-ecosystem in v1.0 | Before M5 | (a) Node.js only, (b) Node.js + Python, (c) All 7 ecosystems | Node.js only for v1.0. The plugin architecture supports adding ecosystems later. Depth over breadth. |

---

# Milestone Summary Timeline

| Milestone | Phase | Complexity | What Ships |
| --- | --- | --- | --- |
| M0: Project Scaffolding | 1 | S | Extension skeleton, Chat Participant echoes input |
| M1: Mock Infrastructure | 1 | S | Mock LM API, model resolver |
| M2: Intent Classifier | 1 | S | Two-tier intent classification |
| M3: Workflow Engine Core | 1 | M | State machine, sequential execution, retry/escalation |
| M4: Agent Spawner | 1 | M | Subagent creation, prompt building, tool scoping |
| M5: Project Model | 1 | M | Node.js project analysis, SQLite persistence |
| M5.5: Wire-Up Spike | 1 | S | End-to-end data flow validation |
| M6: Bug Fix Workflow | 1 | L | **v0.1 — First end-to-end workflow** |
| M7: Basic File Generation | 1 | S | `copilot-instructions.md`, `AGENTS.md` |
| M8: Passthrough Mode | 1 | S | Context-enriched general chat |
| M9: Feature Workflow | 1 | L | Plan approval, parallel layer agents |
| M10: Refactor Workflow | 1 | M | Characterization tests, incremental refactoring |
| M11: Review Workflow | 1 | M | 5-pass parallel code review |
| M12: Remaining Workflows | 1 | M | Documentation, dependency, onboarding + no-tests fallback |
| M13: Config + Polish | 1 | S | Settings, commands, marketplace prep |
| M14: Marketplace Publish | 1 | S | **v0.5 — Preview on VS Code Marketplace** |
| M15: File Watcher | 1.5 | S | Incremental model updates on file change |
| M16: Auto File Regen | 1.5 | M | Automatic file regeneration, edit preservation |
| M17: Pattern Detection | 1.5 | M | Coding pattern detection, deep model |
| M18: Learning Database | 1.5 | M | Snapshots, workflow history, edit tracking |
| M19: Extended File Gen | 1.5 | M | Full file generation catalog |
| M20: Sidebar View | 1.5 | S | **v1.0 — Product complete for solo devs** |
| M21: MCP Scaffold | 2 | S | MCP server lifecycle |
| M22: MCP Project Tools | 2 | S | Project info + context via MCP |
| M23: MCP Workflow Tools | 2 | M | Workflow execution + file gen via MCP |