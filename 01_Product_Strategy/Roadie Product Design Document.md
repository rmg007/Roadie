# Roadie — Product Design Document

**CONFIDENTIAL** · Version 1.0 · April 2026

---

# Product Vision

Roadie is an invisible, autonomous VS Code extension that transforms GitHub Copilot from a reactive code-completion tool into a proactive, workflow-driven engineering partner. It sits between the developer and Copilot as a transparent orchestration layer: intercepting natural language intent from chat, classifying tasks into structured workflows, and executing sophisticated multi-step operations — bug fixing, feature development, refactoring, code review, documentation, dependency management — while the developer experiences nothing more than an unusually capable chat interface. Simultaneously, Roadie maintains a persistent internal model of the project by watching the file system and analyzing code structure, using this model to silently generate and maintain AI configuration files (.github/[copilot-instructions.md](http://copilot-instructions.md), custom agents, skills, hooks, workflows) so that every Copilot interaction — whether through Roadie or not — is precisely tuned to the project's actual technology stack, coding patterns, and architectural decisions. The developer installs the extension and encounters no setup wizard, no configuration screen, no onboarding flow. The first time they chat with Copilot after installation, responses are noticeably more accurate, context-aware, and complete. That moment of quiet surprise — "wait, how did it know that?" — is the product's first impression and its enduring promise: intelligence that compounds over time, works entirely on the developer's machine, requires zero configuration, and makes every AI tool that touches the repository smarter by generating portable, standards-based configuration files that travel with the code.

---

# User Experience Narrative

## Day Zero: Installation

Maya is a solo developer building a SaaS application with Next.js, Prisma, and PostgreSQL. She installs Roadie from the VS Code Marketplace. There is no activation prompt, no welcome tab, no configuration wizard. The extension activates silently. A single, brief notification appears in the status bar: "Roadie active." It fades. Nothing else changes.

## First Interaction: The Quiet Difference

Maya opens the Copilot chat panel and selects "Roadie" from the agent dropdown — the same place she'd pick @workspace or @terminal. She types: "The user profile page is throwing a 500 error after the last deploy." She expects what she always gets from Copilot: a generic suggestion to check her error logs. Instead, Roadie's intent classifier identifies this as a bug-fix trigger. The bug-fix workflow activates. In the chat, she sees progress indicators — "Locating error source...", "Analyzing stack trace...", "Generating fix..." — and then a complete response: the root cause (a null reference in the profile serializer introduced by a recent migration), a code fix applied to the correct file, a verification step that ran her test suite (which passed), a scan for similar patterns elsewhere in the codebase (two more found, both fixed), and a regression test added to prevent recurrence. The entire exchange took ninety seconds. Maya did not navigate to a single file, did not read a stack trace, did not run a test manually. She typed one sentence and got a complete resolution.

## The Background Awakening

While Maya worked, Roadie's project analyzer was lazily building its internal model. Not all at once — it gathered what the bug-fix workflow needed: Next.js 14 with App Router, Prisma ORM with PostgreSQL, Vitest for testing, pnpm as the package manager. It wrote this context into .github/[copilot-instructions.md](http://copilot-instructions.md), noting the tech stack, the testing conventions, and the project structure. It generated a .github/agents/[debugger.agent.md](http://debugger.agent.md) file tuned to this stack. These files appeared as unstaged changes in Maya's Source Control view. She glanced at them, saw they were reasonable, and committed them. She didn't have to.

## Daily Use: The Invisible Partner

Over the next week, Maya uses Roadie without thinking about it. She types "Add a dark mode toggle to the settings page" and gets a feature-development workflow: Roadie presents a plan (database flag in user preferences, API endpoint, React component with system-preference detection, CSS variable theming), waits for her approval, then executes all layers in parallel — she sees progress for each track and the consolidated result. She types "Refactor the authentication module — it's gotten messy" and gets a refactoring workflow that writes characterization tests first, refactors incrementally, and verifies tests pass after each step. She types "Review my changes before I push" and gets a multi-perspective code review covering security, performance, code quality, and test coverage.

Each time a workflow runs, the project model deepens. Roadie learns that Maya uses barrel exports, prefers early returns, follows a specific commit message convention. It updates the generated instruction files to reflect these patterns. When Maya makes a change to a generated file — editing a rule she disagrees with, adding a convention Roadie missed — the edit is preserved on subsequent regenerations. The tool adapts to her; she never adapts to it.

## The Compounding Effect

After a month, Maya's .github/ directory contains a rich set of AI configuration files that encode her project's intelligence. When she uses Copilot's inline completions (not through Roadie), they're better — because Copilot reads [copilot-instructions.md](http://copilot-instructions.md). When she tries Claude Code on a branch, it reads [AGENTS.md](http://AGENTS.md) and understands the project's conventions immediately. The intelligence Roadie generated is portable. It outlives the extension. If Maya uninstalls Roadie tomorrow, the generated files remain, and every AI tool that touches her repository still benefits from them.

---

# Workflow Catalog

Every workflow in Roadie follows a common lifecycle: trigger classification, step execution with model selection, branching and escalation on failure, and structured exit. This section specifies each workflow in full.

## Model Selection Strategy

All workflows use the VS Code Language Model API. The model resolver maps tiers to available models at runtime via `vscode.lm.selectChatModels()`, filtered by cost tier. Specific model names below are examples of what may be available — the actual model catalog changes over time and varies by subscription tier. The escalation hierarchy is cost-driven:

| Tier | Models (examples) | Cost | Usage |
| --- | --- | --- | --- |
| Tier 0 (Free) | GPT-5 mini, GPT-4.1 | 0× | Default for all steps |
| Tier 1 (Standard) | Claude Sonnet 4.6, GPT-5.2, Gemini 2.5 Pro | 1× | Escalation on quality failure |
| Tier 2 (Premium) | Claude Opus 4.6 | 3× | Hard problems, final escalation |

Workflows always start with Tier 0. Escalation occurs when a step produces output that fails validation (tests don't pass, fix doesn't resolve the error, review finds critical issues). Maximum three attempts per step before reporting failure to the developer. If premium quota is exhausted, workflows fall back to included models and note any quality limitations.

> **Design constraint:** All workflows must be viable within Copilot Pro (300 premium requests/month). Cost-awareness is structural, not optional.
> 

## Bug Fix Workflow

**Trigger:** Intent classifier detects error/bug language ("fix," "bug," "error," "broken," "crashing," "500 error," stack traces, error messages).

**User interaction:** None. Fully autonomous.

| Step | Action | Model Tier | Tools | Exit Condition |
| --- | --- | --- | --- | --- |
| 1 | Locate error source | Tier 0 | File search, grep, git log | Error location identified |
| 2 | Diagnose root cause | Tier 0 | Code reading, stack analysis | Cause hypothesis formed |
| 3 | Generate and apply fix | Tier 0 → 1 | Code edit, file write | Fix compiles / lints clean |
| 4 | Verify fix | N/A | Shell: test runner | Tests pass (default 5 min, configurable via `roadie.testTimeout`) |
| 5 | Scan for sibling bugs | Tier 0 | Pattern search, grep | Similar patterns cataloged |
| 6 | Fix siblings (if found) | Tier 0 | Code edit | All siblings resolved |
| 7 | Add regression guard | Tier 0 | Test file creation | New test passes |
| 8 | Generate summary | Tier 0 | N/A | Report delivered to chat |

**Escalation logic:** If Step 4 fails (tests don't pass), the workflow returns to Step 3 with the test output as additional context and escalates to Tier 1. On the second retry, diagnostic logging is added to the failing code path before re-attempting the fix. On the third failure, the workflow escalates to Tier 2 (Opus). If all three attempts fail, the workflow reports the diagnosis, attempted fixes, and test output to the developer with a recommendation.

**No-tests handling:** If the project has no test runner or no tests, Step 4 is replaced with a manual verification note in the summary. Step 7 still creates a test file, bootstrapping the project's test suite.

## Feature Development Workflow

**Trigger:** Intent classifier detects feature/build language ("add," "build," "create," "implement," "new feature," "dark mode," "CRUD for X").

**User interaction:** Plan approval (Step 2). This is the single human-in-the-loop point. All other steps are autonomous.

| Step | Action | Model Tier | Tools | Exit Condition |
| --- | --- | --- | --- | --- |
| 1 | Analyze requirements | Tier 0 | Code reading, project model | Requirements structured |
| 2 | Present plan for approval | N/A | Chat UI: stream.button() | Developer approves or revises |
| 3 | Delegate to layer agents | Tier 0 → 1 | Parallel LLM calls | All layers complete |
| 4 | Integrate layers | Tier 0 | Code edit, imports | Integration compiles |
| 5 | Run tests | N/A | Shell: test runner | Tests pass |
| 6 | Quality review passes | Tier 1 | Security, perf, a11y checks | No critical findings |
| 7 | Generate commit messages | Tier 0 | Git staging | Commit-ready |

**Plan approval mechanics:** The plan is rendered as a Markdown chat message with two `stream.button()` actions: "Approve Plan" and "Revise Plan." Mechanically, the Chat Participant registers two VS Code commands (`roadie.approvePlan` and `roadie.revisePlan`). The workflow step creates a Promise and stores its resolve function. `stream.button()` renders buttons pointing to these commands. When the developer clicks, the command handler resolves the Promise with the user's choice, and the workflow resumes. If the developer selects Revise, the workflow loops to Step 1 with the revision feedback. If the chat session is closed before the developer responds, the Promise is rejected and the workflow is cancelled. There is no timeout — the workflow pauses indefinitely until the developer responds.

**Parallel delegation (Step 3):** Layer-specific agents (database, backend, frontend) run concurrently via `Promise.allSettled()` on the Language Model API. The user sees progress indicators for each track ("Database schema updating...", "Backend endpoints generating...", "Frontend components scaffolding..."). If one track fails, others continue; the failed track retries independently.

## Refactoring Workflow

**Trigger:** Intent classifier detects refactoring language ("refactor," "clean up," "restructure," "simplify," "extract," "messy").

**User interaction:** None. Fully autonomous.

| Step | Action | Model Tier | Tools | Exit Condition |
| --- | --- | --- | --- | --- |
| 1 | Analyze current structure | Tier 0 | Code reading | Structure mapped |
| 2 | Write characterization tests | Tier 0 → 1 | Test file creation, test runner | Tests capture current behavior |
| 3 | Refactor incrementally | Tier 0 | Code edit | Each change compiles |
| 4 | Verify after each step | N/A | Shell: test runner | Characterization tests pass |
| 5 | Generate before/after summary | Tier 0 | Diff analysis | Summary delivered |

**Key invariant:** Public APIs are never changed. If a refactoring would alter a public interface, the workflow reports this and stops, asking for explicit approval.

**Incremental safety:** Steps 3 and 4 loop together. Each incremental change (extract function, rename, restructure) is followed by a test run. If tests fail, the change is reverted and the workflow attempts an alternative approach. This ensures no single refactoring step breaks the codebase.

## Code Review Workflow

**Trigger:** Intent classifier detects review language ("review," "check my code," "before I push," "look over," "any issues").

**User interaction:** None. Fully autonomous.

The code review workflow runs multiple focused passes rather than a single monolithic review. Each pass uses a different prompt configuration (agent role) to ensure depth:

| Pass | Perspective | Focus Areas | Model | Output |
| --- | --- | --- | --- | --- |
| 1 | Security Reviewer | OWASP Top 10, injection, auth, secrets | Tier 1 | Security findings |
| 2 | Performance Reviewer | N+1 queries, re-renders, memory leaks, complexity | Tier 0 | Performance findings |
| 3 | Code Quality Reviewer | Duplication, complexity, naming, patterns | Tier 0 | Quality findings |
| 4 | Test Coverage Reviewer | Untested paths, edge cases, assertion quality | Tier 0 | Coverage findings |
| 5 | Standards Reviewer | Project conventions (from project model) | Tier 0 | Standards findings |

All passes run concurrently via `Promise.allSettled()`. Results are consolidated into a single structured review with findings categorized as Critical, Warning, or Suggestion. The review targets the current git diff (staged or unstaged changes) unless the developer specifies a broader scope.

## Documentation Workflow

**Trigger:** Intent classifier detects documentation language ("document," "README," "API docs," "explain this module," "update docs").

| Step | Action | Model Tier | Tools | Exit Condition |
| --- | --- | --- | --- | --- |
| 1 | Read actual code | Tier 0 | Code reading | Code understood |
| 2 | Generate documentation | Tier 0 | File creation | Docs written |
| 3 | Cross-reference existing docs | Tier 0 | File reading, diff | Drift identified |
| 4 | Reconcile and deliver | Tier 0 | File edit | Docs match reality |

The documentation workflow reads the actual implementation — never existing documentation — as its source of truth. It then checks existing docs for drift (claims that no longer match the code) and either updates them or flags discrepancies.

## Dependency Management Workflow

**Trigger:** Intent classifier detects dependency language ("update deps," "security audit," "vulnerabilities," "outdated packages," "upgrade").

| Step | Action | Model Tier | Tools | Exit Condition |
| --- | --- | --- | --- | --- |
| 1 | Audit dependency tree | Tier 0 | Shell: npm/pip audit | Tree mapped |
| 2 | Identify CVEs | Tier 0 | Audit output parsing | Vulnerabilities listed |
| 3 | Check for breaking changes | Tier 0 → 1 | Changelog analysis | Breaking changes flagged |
| 4 | Produce upgrade plan | Tier 0 | Risk ordering | Plan generated |
| 5 | Execute upgrades with testing | N/A | Shell: install + test | Tests pass per upgrade |

Upgrades in Step 5 are applied one at a time in risk order (lowest risk first). After each upgrade, the test suite runs. If tests fail, the upgrade is reverted and flagged in the report. The developer receives a final summary showing which upgrades succeeded, which were reverted, and which require manual intervention.

## Onboarding Workflow

**Trigger:** Intent classifier detects onboarding language ("explain this project," "walk me through," "what does this codebase do," "where do I start").

| Step | Action | Model Tier | Tools | Exit Condition |
| --- | --- | --- | --- | --- |
| 1 | Generate architecture overview | Tier 0 | Project model, code reading | Architecture described |
| 2 | Identify key patterns | Tier 0 | Pattern analysis | Patterns documented |
| 3 | Map entry points | Tier 0 | Config/route analysis | Entry points listed |
| 4 | Suggest starter tasks | Tier 0 | Git log, issue analysis | Tasks recommended |

This workflow uses the project model extensively and typically triggers its construction if the model is not yet built. Results are delivered as a structured chat response, not as generated files, since the output is ephemeral and developer-specific.

---

# Agent Architecture

Roadie uses the term "agent" to describe role-specific prompt configurations, not separate VS Code Chat Participants. There is one Chat Participant — Roadie itself — registered via the VS Code Chat Participant API. All internal "agents" are an abstraction: a combination of a system prompt template, a scoped tool set, and a model-tier preference. The developer never sees or interacts with individual agents directly.

## The Chat Participant: Roadie

The developer selects Roadie from the VS Code chat agent dropdown (the same dropdown where @workspace and @terminal appear). Once selected, all prompts flow through Roadie's Chat Participant handler. The handler has two responsibilities: intent classification (routing to workflows) and passthrough (forwarding non-workflow prompts to the underlying model with enhanced context from the project model).

> **Important:** Roadie does NOT invisibly intercept Copilot's default agent. The developer makes an explicit selection once, then operates naturally. The invisibility is in the execution, not the invocation.
> 

## Intent Classifier

The intent classifier is the first component in the Chat Participant handler. It receives the developer's raw prompt and produces a classification: which workflow to trigger, or "passthrough" for general questions. The classifier uses a two-tier approach. Tier 1 is a local keyword/regex classifier that runs instantly with zero cost. If local confidence is below 0.7, Tier 2 uses the first LLM call (piggybacked on the response, not a separate call) to classify with structured output. Classification latency target: under 500ms (under 10ms for local tier).

If the prompt does not match any workflow (general questions, casual chat, code explanations), Roadie passes the prompt to the underlying model with the project model injected as context. This means even non-workflow prompts benefit from Roadie's project awareness — the developer gets better answers than bare Copilot because the context is richer.

## Internal Agent Roles

Each agent role is defined by three components: a system prompt template, a set of tools it can invoke, and a preferred model tier. Agents are instantiated by the workflow engine as needed and destroyed when the workflow completes.

| Agent | Role | Tools | Tier | Lifecycle |
| --- | --- | --- | --- | --- |
| Diagnostician | Locate and diagnose errors | File search, grep, git log, stack trace parser | 0 | Ephemeral |
| Fixer | Generate and apply code fixes | Code edit, file write, linter | 0 → 2 | Ephemeral |
| Planner | Create execution plans for features | Project model, code reading | 0 → 1 | Ephemeral |
| Database Agent | Schema changes, migrations, queries | ORM analysis, migration tools | 0 → 1 | Ephemeral |
| Backend Agent | API endpoints, business logic, services | Route analysis, code edit | 0 → 1 | Ephemeral |
| Frontend Agent | UI components, state, styling | Component analysis, code edit | 0 → 1 | Ephemeral |
| Refactorer | Incremental code restructuring | Code edit, test runner, diff | 0 | Ephemeral |
| Security Reviewer | OWASP analysis, secret detection | Pattern matching, dependency scan | 1 | Ephemeral |
| Performance Reviewer | N+1, complexity, memory analysis | Code reading, pattern matching | 0 | Ephemeral |
| Quality Reviewer | Duplication, naming, style analysis | Code reading, project conventions | 0 | Ephemeral |
| Test Reviewer | Coverage gaps, edge cases, assertion quality | Test file reading, coverage data | 0 | Ephemeral |
| Standards Reviewer | Project convention compliance | Project model, convention rules | 0 | Ephemeral |
| Documentarian | Generate and update documentation | Code reading, file creation | 0 | Ephemeral |
| Project Analyzer | Build and maintain project model | File system, dependency manifests, git | 0 | Persistent |

**Persistent vs Ephemeral:** All agents are ephemeral (created for a workflow, destroyed after) except the Project Analyzer, which persists across the extension's lifecycle. The Project Analyzer is not invoked through chat — it runs in response to file-system events and workflow requests for project context.

## Agent Implementation

Each agent is implemented as a TypeScript class with the following interface:

- **systemPrompt:** A template string with placeholders for project context (tech stack, conventions, file paths). Populated from the project model at invocation time.
- **tools:** A whitelist of tool identifiers the agent can use. Tools include file read/write, shell execution, grep/search, git operations, and project model queries.
- **modelPreference:** The starting tier and maximum tier for this agent. The workflow engine handles escalation.
- **validate(output):** A function that determines whether the agent's output meets quality criteria. If validation fails, the workflow engine escalates the model tier and retries.

---

# Project Model Specification

The project model is Roadie's internal representation of the developer's project. It powers context injection for all workflows and drives file generation. The model is built lazily — populated as workflows request context — and persisted to a local SQLite database for continuity across sessions.

## Phase 1: Shallow Model

The Phase 1 model contains enough information to power accurate workflow execution and generate useful instruction files.

| Category | Data Points | Source |
| --- | --- | --- |
| Tech Stack | Language(s), framework(s), version(s) | package.json, pyproject.toml, go.mod, Cargo.toml, pom.xml, *.csproj, Gemfile |
| Runtime | Node version, Python version, runtime constraints | Engine fields, .node-version, .python-version, runtime config |
| Package Manager | npm, pnpm, yarn, pip, cargo, go, maven, bundler | Lock files (pnpm-lock.yaml, yarn.lock, etc.) |
| Test Runner | Vitest, Jest, pytest, go test, cargo test | Config files, package.json scripts |
| Linter / Formatter | ESLint, Prettier, Black, Ruff, gofmt | Config files (.eslintrc, .prettierrc, etc.) |
| ORM / Database | Prisma, TypeORM, SQLAlchemy, Django ORM | Dependency manifests, config files |
| Directory Structure | Source root, test root, config root, output directories | File system analysis, framework conventions |
| Build / Run Commands | Build, dev, test, lint, format commands | package.json scripts, Makefile, pyproject.toml |

> **Scope note:** v1.0 implements the Node.js/TypeScript analyzer with full support for all data points above. Other ecosystems are detected at a basic level (language and primary framework from dependency manifest) but do not receive deep analysis. The AnalyzerPlugin architecture supports adding ecosystem-specific analyzers post-v1.0. Detection is generic, not curated. Roadie reads dependency manifests for whatever ecosystem it finds. JS/TS will have the deepest support; other ecosystems are functional but less nuanced.
> 

## Phase 1.5: Deep Model Additions

| Category | Data Points | Source |
| --- | --- | --- |
| Module Dependencies | Which files import which (inter-module relationships) | Import statement analysis via regex (not AST) |
| Coding Patterns | Naming conventions, export style, error handling, async patterns | Sampling actual source files (10–20 representative files) |
| Git Patterns | Commit message conventions, branching strategy, active areas | git log analysis, recent commit history |
| Developer Preferences | Edits to generated files, workflow outcome patterns | Edit tracking (opt-in), workflow history (opt-in) |

## Schema (SQLite)

The project model is stored in `.github/.roadie/project.db` with the following core tables:

- **project_meta:** Key-value pairs for top-level project attributes (name, root path, detected stack, last analyzed timestamp).
- **dependencies:** Package name, version, dev/prod flag, ecosystem. Populated from manifest parsing.
- **directory_map:** Path, role (source/test/config/output/static), framework association. Populated from directory analysis.
- **commands:** Name, command string, source file, type (build/test/dev/lint/format). Populated from scripts analysis.
- **patterns:** (Phase 1.5) Pattern type, description, evidence (file paths and line numbers), confidence score.
- **module_graph:** (Phase 1.5) Source file, imported file, import type (named/default/namespace).
- **edit_history:** (Phase 1.5, opt-in) File path, section ID, previous hash, current hash, timestamp.

## Model Lifecycle

**Construction:** Lazy. The model is not built at install time. It is populated when a workflow first requests context (typically the first chat interaction). The Project Analyzer gathers only what the requesting workflow needs, then expands opportunistically if time permits.

**Maintenance:** In Phase 1, the model is rebuilt on-demand when workflows request context and the existing data is stale (determined by comparing file modification timestamps against the last analysis timestamp). In Phase 1.5, the VS Code FileSystemWatcher triggers incremental model updates when dependency manifests, configuration files, or source directory structures change.

**Persistence:** The SQLite database persists across VS Code sessions. It is stored in `.github/.roadie/project.db` and is .gitignored by default.

---

# File Generation Rules

Roadie generates and maintains a set of AI configuration files that encode project intelligence in a portable, standards-based format. These files are useful to any AI tool that reads them — Copilot, Claude Code, Gemini CLI, and others — regardless of whether Roadie is installed.

## Generated Files

| File | Purpose | Creation Trigger |
| --- | --- | --- |
| .github/[copilot-instructions.md](http://copilot-instructions.md) | Project-wide Copilot context: tech stack, conventions, patterns | First workflow execution (project model populated) |
| .github/instructions/*.[instructions.md](http://instructions.md) | Per-language/framework-specific rules | Workflow encounters language-specific patterns |
| .github/agents/*.[agent.md](http://agent.md) | Specialized agent definitions (debugger, reviewer, etc.) | Corresponding workflow first executes |
| .github/skills/*/[SKILL.md](http://SKILL.md) | Procedural skills (deployment, migration, testing procedures) | Workflow discovers repeatable procedures |
| .github/hooks/*.json | Lifecycle automation (format on save, lint on edit) | Formatter/linter detected in project |
| .github/workflows/*.yml | CI/CD workflow templates | Project structure suggests CI needs |
| [AGENTS.md](http://AGENTS.md) (root) | Cross-tool instructions (Claude Code, Codex, Gemini CLI) | First workflow execution |
| .github/.roadie/project.db | Project model database (gitignored) | First workflow execution |
| .github/.roadie/.gitignore | Gitignore for Roadie internal files | First workflow execution |

## Generation Lifecycle

**Lazy, not eager:** No files are generated at install time. Files are created when workflows produce context that warrants them. This ensures files reflect actual observed project state, not guesses.

**Event-driven updates:** In Phase 1, files are regenerated when a workflow runs and detects that the project model has changed since the last generation. In Phase 1.5, the file watcher triggers regeneration when relevant source files change.

**Minimal writes:** Roadie only writes a file when its content would actually change. A hash of each generated file's content is stored in the project database; if regeneration produces identical content, no write occurs.

## Section Ownership Model

Generated files use HTML comment markers to delineate ownership. This allows developers to edit generated files without losing their changes on regeneration.

**Marker format:** `<!-- roadie:start:{section-name} -->` and `<!-- roadie:end:{section-name} -->`. Content between markers is owned by Roadie. Content outside markers is owned by the developer.

**Regeneration behavior:**

- **Content outside Roadie markers:** Never touched. Developer additions, edits, and deletions are fully preserved.
- **Content inside Roadie markers, unmodified by developer:** Updated freely. This is the normal case — Roadie regenerates its own sections.
- **Content inside Roadie markers, modified by developer:** Appended below existing content, separated by `<!-- roadie:merged:{timestamp} -->`. Roadie stores a hash of the last-generated content per section. If the current on-disk content doesn't match the stored hash, a human edited it. In this case, Roadie appends the new generated content below the existing human-edited content, never overwriting it.
- **Section markers removed by developer:** The entire file is considered human-owned. Roadie logs a warning and does not modify the file.

> **Principle:** Roadie never silently overwrites human work. If a developer edits a generated file, their edits are the highest-priority content. Roadie adapts around them.
> 

## Git Integration

By default, generated files are written to disk with no staging and no committing. They appear as unstaged changes in VS Code's Source Control view. The developer decides when and how to commit them.

If the `autoCommit` configuration option is enabled (default: false), Roadie stages and commits generated files with the message `chore(roadie): update AI configuration [skip ci]`. Roadie never amends existing commits. Each generation cycle that produces changes creates a new, separate commit.

---

# Configuration Model

Roadie requires zero configuration. Every feature works out of the box with sensible defaults. Configuration exists solely as opt-in overrides for developers who want finer control. All configuration is stored in `.vscode/settings.json` under the `roadie` namespace.

| Setting | Type | Default | Effect When Enabled |
| --- | --- | --- | --- |
| roadie.telemetry | boolean | false | Sends anonymous, aggregate usage patterns (workflow types, model tiers used, success/failure rates). Never sends code, file names, file content, or project details. |
| roadie.editTracking | boolean | false | Tracks developer edits to Roadie-generated files. Stores diffs in the local database. Used to learn preferences and improve future generation. |
| roadie.workflowHistory | boolean | false | Persists workflow outcomes (which workflows ran, success/failure, escalation patterns). Used for pattern learning and self-optimization. |
| roadie.modelPreference | enum / null | null | Overrides the default model tier. Values: "economy" (Tier 0 only), "balanced" (default behavior), "quality" (start at Tier 1). Does not override escalation logic. |
| roadie.autoCommit | boolean | false | Stages and commits generated .github/ files automatically with conventional commit messages. Never amends existing commits. |
| roadie.testTimeout | number | 300 | Maximum seconds to wait for test suite execution before timing out. Applies to all workflows that invoke tests. |
| roadie.testCommand | string / null | null | Overrides the auto-detected test command. Use when Roadie detects the wrong test runner or the test command requires specific flags. |

> **Design principle:** Every boolean default is false. Roadie does nothing the developer hasn't implicitly or explicitly consented to. The extension is fully functional with all settings at their defaults.
> 

---

# Privacy and Data Model

Roadie is designed around a local-first, privacy-preserving data model. No account is required, no cloud service is contacted for core functionality, and the extension works fully offline.

## Data Storage

| Data | Location | Gitignored | Retention |
| --- | --- | --- | --- |
| Project model (SQLite) | .github/.roadie/project.db | Yes | Persists across sessions; rebuilt on demand |
| Section hashes | .github/.roadie/project.db | Yes | Updated each generation cycle |
| Edit tracking (opt-in) | .github/.roadie/project.db | Yes | Accumulated; no automatic pruning |
| Workflow history (opt-in) | .github/.roadie/project.db | Yes | Accumulated; no automatic pruning |
| Generated AI config files | .github/, [AGENTS.md](http://AGENTS.md) | No | Persist intentionally — useful without Roadie |
| Extension state | VS Code globalState | N/A | Managed by VS Code extension lifecycle |

## What Is Never Stored

- Application source code (Roadie reads code but never copies it into its database)
- Chat conversation history (handled by VS Code's chat infrastructure)
- Personal information, credentials, or secrets
- File contents beyond what the project model schema requires (dependency names, directory paths, command strings)

## What Is Never Transmitted

- Code content, file names, or project details — even with telemetry enabled
- Any data to any server controlled by Roadie. Roadie has no server.
- LLM prompts go through the VS Code Language Model API to the developer's Copilot subscription. Roadie does not proxy, log, or intercept these payloads beyond the current session.

## Telemetry (Opt-In Only)

If `roadie.telemetry` is enabled, the following aggregate, anonymous data is transmitted:

- Workflow types triggered (e.g., "bug-fix" ran 3 times this week)
- Model tiers used (e.g., Tier 0: 80%, Tier 1: 15%, Tier 2: 5%)
- Success/failure rates per workflow type
- Extension version and VS Code version

This data contains no code content, no file names, no project identifiers, and no information that could identify the developer or their project.

## Uninstall Behavior

When Roadie is uninstalled, the extension cleans up its background processes and VS Code-managed state. The `.github/.roadie/` directory (containing project.db) can be safely deleted by the developer. Generated AI configuration files (.github/[copilot-instructions.md](http://copilot-instructions.md), agents, skills, etc.) are intentionally left in place — they are useful without the extension and carry no dependency on Roadie.

---

# Scope Boundaries

Roadie has explicit boundaries that define what it is not, what it does not do, and what it defers to other tools. These boundaries are product decisions, not limitations.

## Roadie Does Not Replace

- **VS Code:** Roadie is an extension within VS Code, not an alternative to it.
- **GitHub Copilot:** Roadie enhances Copilot by providing better context and structured workflows. It relies on Copilot's Language Model API for all LLM calls.
- **Deterministic tools:** Formatters (Prettier, Black), type checkers (tsc, mypy), and test runners (Vitest, pytest) remain standalone. Roadie integrates them via hooks and shell commands, never reimplements them.
- **Version control:** Roadie generates files but does not manage git operations beyond optional auto-commit of its own generated files.

## Roadie Does Not Do

- Run its own LLM. All model calls go through the VS Code Language Model API, using the developer's Copilot subscription.
- Require an account, cloud service, or internet connection for core functionality.
- Modify application code outside of active workflow execution. The only files Roadie writes autonomously are in .github/ and root AI config files ([AGENTS.md](http://AGENTS.md)).
- Generate files eagerly at install time. All generation is lazy and event-driven.
- Handle monorepos in v1. If a monorepo is detected, Roadie operates at the root level and notes the limitation.
- Target teams or enterprises. v1 is designed for solo developers. Multi-user coordination, shared configuration, and team-level analytics are future phases.
- Intercept Copilot's default chat agent invisibly. The developer explicitly selects Roadie from the agent dropdown.

## Agent-Based Tool Replacement

Roadie replaces certain standalone tools with agent-based equivalents that can fix problems, not just report them:

| Replaced Tool | Roadie Equivalent | Key Difference |
| --- | --- | --- |
| GitLeaks (secret scanning) | PreToolUse hooks + scanning skill | Detects secrets AND suggests remediation |
| npm audit / pip-audit | Dependency management workflow | Audits AND executes upgrades with test verification |
| Manual code review | Code review workflow (5-pass) | Multi-perspective review with project-specific rules |
| Manual doc auditing | Documentation workflow drift detection | Detects drift AND reconciles docs with code |

---

# Success Metrics

These metrics measure whether Roadie is delivering on its product promise. They are product quality metrics, not business metrics. All metrics are measurable locally or through opt-in telemetry.

## Primary Metrics

| Metric | Definition | Target |
| --- | --- | --- |
| Workflow completion rate | Percentage of triggered workflows that complete successfully without developer intervention (excluding the one approval point in feature development) | ≥85% within 30 days of install |
| First-attempt fix rate | Percentage of bug-fix workflows that resolve the issue on Tier 0 without escalation | ≥60% |
| Model cost efficiency | Percentage of LLM calls made at Tier 0 across all workflows | ≥70% Tier 0 calls |
| Time-to-value | Time from extension install to first successful workflow completion | <5 minutes (first chat interaction) |
| Generated file accuracy | Percentage of generated instruction file content that the developer does not edit within 7 days | ≥90% unedited |

## Secondary Metrics

| Metric | Definition | Target |
| --- | --- | --- |
| Passthrough quality lift | Developer-perceived improvement in non-workflow Copilot responses when Roadie context is injected vs. bare Copilot | Measurable improvement (qualitative) |
| Escalation efficiency | Percentage of escalated workflows (Tier 0 → 1 or 1 → 2) that succeed after escalation | ≥80% of escalations resolve |
| Test generation adoption | When Roadie generates tests (in no-test projects), percentage that the developer retains | ≥70% retained |
| Project model staleness | Average age of project model data when a workflow uses it (lower is better) | <24 hours in Phase 1.5 |
| Edit preservation accuracy | When a developer edits a generated file and Roadie regenerates, percentage of human edits correctly preserved | 100% (zero data loss) |

## Anti-Metrics (What We Explicitly Do Not Optimize For)

- **Number of files generated:** More files is not better. Roadie should generate the minimum set of files that provide meaningful value. An unused agent definition is worse than no agent definition.
- **LLM call volume:** More calls is not better. Cost efficiency is a first-class design constraint. Fewer, better-targeted calls at lower tiers are preferred.
- **User engagement/interaction frequency:** Roadie's success is measured by how little it demands of the developer, not how much. Fewer interruptions and fewer required interactions indicate a healthier product.
- **Feature count:** Depth of workflow quality matters more than breadth of workflow types. Seven workflows that work flawlessly outperform twenty that are mediocre.