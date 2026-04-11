# Roadie - Project Mission and Origin Story

**Document type:** Root context for all agents and contributors
**Repository:** github.com/rmg007/Roadie
**Last updated:** April 2026
**Status:** Canonical - read this FIRST before touching any code or documentation

---

## How This Started

On April 2, 2026, a developer uploaded a GitHub Copilot customization reference document to a Claude chat with one request: "improve/enhance/update this reference."

That reference was a personal cheat sheet - how to write copilot-instructions.md, how to configure custom agents, how to create skills and hooks. The kind of document every developer who uses Copilot seriously eventually writes for themselves because the official docs are scattered across VS Code docs, GitHub docs, and blog posts.

The improvement process turned into creation. Instead of polishing one reference, three comprehensive guides were produced:

1. **Copilot Agent Environment Guide** - Decision framework for choosing between Local, Background (CLI), and Cloud agents. When to use each, how they differ, and how to route tasks between them.

2. **Copilot Agents Orchestration Guide** - How to build custom agents, wire them together with handoffs and subagents, create orchestrator patterns, and delegate across environments.

3. **Workspace Optimizer Agent** - A ready-to-use .agent.md file that audits and maintains the .github/ directory. The operational embodiment of best practices for AI configuration.

These three guides distilled hundreds of hours of research into actionable reference documents. They explained everything a developer needs to know to configure their AI tooling properly.

Then the pivotal question was asked: **if we know exactly what good configuration looks like, why make the developer write it?**

---

## The Insight

The guides contain the complete knowledge of how to configure AI tools for any project. A project's package.json, tsconfig.json, source structure, and coding patterns contain everything needed to generate that configuration automatically. The gap between "knowledge of how to configure" and "information about what to configure" is exactly what software fills.

The developer's next message defined the entire product philosophy in a few sentences:

> "The developer's experience in the first 60 seconds after installation is ABSOLUTELY NOTHING. The dev tool installed, configured, and started in silent. Plug and play. Once the developer starts to chat with Copilot, the tool works and does the magic. I want to spoil the developer as much as my tool can do."

That message - and the workflow examples that followed (bug fix chains, feature development with parallel agents, learning from developer edits) - transformed a documentation project into a product design session. Five architecture options were evaluated (pure .agent.md, pure extension, extension + generated files, MCP server only, all of the above). Option 3 (Extension + Generated Files) was chosen because it gives zero-intervention behavior, workflow control in code, learning through state persistence, AND portability through generated files.

**Roadie is those guides turned into software.**

---

## The Mission

**Make every AI coding tool smarter about your project without you doing anything.**

Roadie is an invisible layer that sits between developers and their AI tools. It watches, learns, and silently maintains the configuration files that make AI tools understand your codebase. The developer never thinks about Roadie. They just notice that Copilot gives better answers, Claude Code understands their patterns, and every AI tool they use seems to already know their project.

---

## The Product Philosophy

### Invisible by Default

No setup wizard. No configuration prompts. No "getting started" tutorial. The extension installs, activates silently, and waits. The first time the developer selects @roadie in the chat dropdown and describes a task, Roadie has already analyzed their project, knows their tech stack, understands their patterns, and executes a sophisticated multi-step workflow - all while the developer thinks they're just chatting with Copilot.

### Spoil the Developer

Every workflow exhausts autonomous options before interrupting the human. The bug fix workflow doesn't just find the bug - it fixes it, runs the tests, searches for similar bugs elsewhere, fixes those too, adds regression tests, and presents a clean summary. The developer gets the result, not the process.

### Portable Intelligence

The .github/ files Roadie generates work with every AI tool that reads them - Copilot, Claude Code, Gemini CLI, Cursor. If a developer uninstalls Roadie, the generated configuration stays and continues helping. Roadie's value persists even when Roadie isn't running.

### Zero Configuration, Opt-In Everything

Every boolean setting defaults to false. Edit tracking? Off. Workflow history? Off. Telemetry? Off. The tool works fully with all defaults. Turning things on is an explicit choice. This builds trust - especially for enterprise adoption where security teams kill anything that phones home without consent.

### AI Builds AI Tooling

All Roadie code is written by AI coding agents (Claude Code, Codex). The architecture is optimized for their productivity: modules under 300 lines, explicit TypeScript interfaces at every boundary, comprehensive tests as guardrails, JSDoc headers for context. The documentation was designed to be consumed by AI agents as build instructions - each module has a "Build Prompt" ready to paste.

---

## What Roadie Does

### Phase 1 - Active Mode (The Workflow Engine)

Developer chats with @roadie. A two-tier intent classifier (local regex + LLM double-duty) routes prompts to workflow definitions. A workflow engine executes multi-step state machines, spawning ephemeral subagents for each step with model escalation (free to standard to premium).

**Seven workflows:** Bug fix (8 steps with sibling scanning and regression testing), Feature development (7 steps with plan approval via stream.button() and parallel layer agents), Refactoring (5 steps with inner loop and characterization tests), Code review (5 parallel passes: security, performance, quality, coverage, standards), Documentation, Dependency management, Onboarding.

### Phase 1.5 - Passive Mode (The Invisible Layer)

Roadie watches the file system silently using VS Code's FileSystemWatcher. When dependencies change, the project model updates and files regenerate. When a developer edits a generated file, Roadie preserves their changes using an append-below merge strategy (never overwrites human work).

**Eight file generators** run in parallel (under 2s total): Copilot instructions, path-specific instructions, agent definitions, skill definitions, hooks, workflows, templates, AGENTS.md. All respect section ownership markers.

**Codebase Dictionary:** When workflows generate code, they annotate every entity with structured metadata - what it does, why it was created, what depends on it. Stored in SQLite, queryable by any workflow for context injection.

### Phase 2 - MCP Server (Cross-Tool Intelligence)

A stdio-based MCP server wraps Roadie's capabilities as tools any AI agent can call. Provider interfaces abstract the VS Code API so the same core engine runs inside VS Code or standalone via npx roadie-mcp --project .

**Ten MCP tools:** analyze_project, get_project_context, run_workflow, get_workflow_status, generate_file, generate_all_files, query_patterns, query_workflow_history, get_recommendations, rescan_project.

### Phase 2.5 - Adaptive Learning (Future)

Closes the feedback loop: Roadie observes whether its generated configuration actually helps, and adjusts. Generation quality scoring, adaptive prompt templates, intent classifier weight tuning, workflow step optimization, developer preference learning. All rule-based (not ML), all local, all built on data already collected by the Phase 1.5 edit tracker and learning database. Requires 2-4 weeks of real usage data before building.

### Phase 3 - Teams and Enterprise (Future)

GitHub App distribution, org-level configuration hierarchy, cross-repo consistency engine, multi-user edit conflict resolution, shared skill/agent registry, team dashboard. Explicitly deferred until solo experience is proven.

---

## What's in This Repository

### Agent Entry Points

- 00_START_HERE.md - THE canonical entry point. Reading order, canonical module ID map, archived file list, ground rules. Every agent reads this first.
- AGENT_ENTRYPOINT.md - Execution sequence, context clearing rules, deferred architecture decisions.

### Documentation (8 folders + 2 future phase folders)

```
01_Product_Strategy/          - PDD, roadmap, key decisions, implementation checklist
02_Technical_Architecture/    - TAD, extension manifest, standalone mode, Phase 1/1.5/2 architecture,
                                generator templates, UI/UX spec, core/shell split, issues dashboard
03_Implementation_Specs_Phase_1/ - Module build order (13 steps), project structure, complete master spec
04_Implementation_Specs_Phase_1.5/ - File watcher, section manager, persistence, generators,
                                     edit tracker, learning DB, codebase dictionary
05_Implementation_Specs_Phase_2/  - MCP server master spec
06_Workflows_and_Prompts/     - Intent classification (with exact weights), workflow state machines
                                (all 7), prompt templates for every step
07_Patterns_and_Standards/    - TypeScript interfaces (COPY VERBATIM), Zod schemas (COPY VERBATIM),
                                implementation patterns, model selection, security and migration, UI/UX
08_Integration_and_Testing/   - E2E scenarios, testing strategy, error taxonomy, security baseline
09_Phase_2.5_Adaptive_Learning/ - Adaptive learning spec (design phase, not ready to build)
10_Phase_3_Teams_Enterprise/    - Teams/enterprise vision (deferred, not ready to build)
```

### Quality Assurance

- Issues and Fixes Status Tracking Dashboard - All 22 issues from the Zero-Guesswork audit: ALL RESOLVED
- ARCHIVED_BLOCKING/ - Historical contradiction documents (DO NOT implement from these)
- The documentation passed a formal Zero-Guesswork audit - an autonomous coding agent can build from this corpus with zero clarification pauses

### Source of Truth Hierarchy

```
GitHub repo (this repository)  <-- PRIMARY for coding agents (self-contained, no external deps)
    ^ exported from
Notion (living documents)      <-- Collaborative editing surface for humans
    | implement
Source code (when built)       <-- Must match the specs exactly
```

**For coding agents:** Everything you need is in THIS REPO. Do not fetch Notion pages, external URLs, or web resources.

---

## Architecture in One Paragraph

A VS Code extension registers a Chat Participant (@roadie). A two-tier intent classifier routes prompts to workflow definitions. A workflow engine executes multi-step state machines, spawning ephemeral subagents via the VS Code Language Model API with three-tier model escalation. A project model (SQLite-backed at .github/.roadie/project-model.db, incrementally updated by a file watcher) provides context via toContext() with token budgeting. File generators produce .github/ configuration files using section ownership markers and an append-below merge strategy. A codebase dictionary captures entity metadata at generation time. A learning database tracks edit history and workflow outcomes. An MCP server wraps everything as tools for cross-tool compatibility. The core engine is VS Code-independent (provider abstraction) so the MCP server runs standalone.

---

## Key Technical Decisions (Settled - Do Not Revisit)

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| D1 | Architecture | Extension + Generated Files | Zero-intervention + workflow control + portability |
| D2 | Language | TypeScript everywhere | AI agents produce better TS than Rust; one build system |
| D3 | Development | AI agents write all code | Architecture optimized for AI productivity (300-line modules) |
| D4 | Target user | Solo devs on Copilot Pro/Pro+ | Fast adoption, direct feedback, no procurement |
| D5 | Phasing | Phase 1 to 1.5 to 2 | Each phase independently valuable |
| D6 | File watching | VS Code FileSystemWatcher | Works in Remote Dev (SSH/WSL), unlike chokidar |
| D7 | Merge strategy | Append-below | Both human + Roadie content visible; zero data loss |
| D8 | Config defaults | All booleans false | Roadie does nothing developer hasn't consented to |
| D9 | Database | Single SQLite file | Zero config, no server, better-sqlite3 |
| D10 | Intent classifier | Double-duty pattern | One LLM call serves classification AND response |
| D11 | Source of truth | Notion (living) to GitHub (export) to code | Multiple agents need shared editing surface |

Full rationale: 01_Product_Strategy/Key Decisions and Rationale Log.md

---

## What Roadie is NOT

- **Not a Copilot replacement.** Roadie makes Copilot better. It rides on the developer's existing subscription.
- **Not a code generator.** Roadie generates configuration files. Workflows execute through Copilot's models.
- **Not a cloud service.** Everything runs locally. No accounts, no sign-up, no data leaves the machine unless telemetry is opted into.
- **Not a linter.** Roadie doesn't enforce rules - it teaches AI tools what the rules are.
- **Not a team tool (yet).** Solo developers only in v1.0. Teams/enterprise is Phase 3+ - explicitly deferred.

---

## Current Status (April 2026)

| Phase | Documentation | Code | Status |
|-------|--------------|------|--------|
| Phase 1 (Active Mode) | Complete (14 modules, 13-step build order) | Not started | Ready to build |
| Phase 1.5 (Passive Mode) | Complete (11 spec pages + codebase dictionary) | Not started | Ready to build after Phase 1 |
| Phase 2 (MCP Server) | Complete (7 spec pages) | Not started | Ready to build after Phase 1.5 |
| Phase 2.5 (Adaptive Learning) | Design phase | Not started | Requires 2-4 weeks usage data |
| Phase 3 (Teams/Enterprise) | Vision only | Not started | Deferred until solo value proven |

**Documentation readiness:** A Zero-Guesswork audit identified 14 gaps. All 22 tracked issues have been resolved. The corpus has passed Mission-Ready standard.

**Next step:** Begin Phase 1 Module Build Order (13 steps, ~33.5 hours estimated for AI agents).

---

## For Agents Reading This

You're about to build Roadie. Here's how:

### Reading Order
1. **00_START_HERE.md** - the canonical entry point (start here, not this file)
2. **07_Patterns_and_Standards/Shared TypeScript Interfaces.md** - COPY VERBATIM
3. **07_Patterns_and_Standards/Shared Zod Schemas.md** - COPY VERBATIM
4. **03_Implementation_Specs_Phase_1/Module Build Order and Verification.md** - execute Build Prompts literally

### Rules
- **Every design decision is already made.** Implement the spec, don't redesign.
- **If something is ambiguous, stop and ask.** Don't guess, don't improvise.
- **Run tests after every module.** The specs include verification criteria.
- **Keep files under 300 lines.** Split if needed - this is a hard rule.
- **All cross-module inputs validated with Zod.** No exceptions.
- **All Phase 1.5 file I/O must be async.** Never use fs.readFileSync in watcher/generator paths.
- **Never use child_process.exec().** Always spawn() with argv array.
- **Copy types verbatim from the Shared TypeScript Interfaces.** Do not rename, reformat, or add fields.
- **No external URLs.** Everything you need is in this repository.
- **Prohibited libraries:** chokidar, react, electron, axios.

### The Goal
The goal is not clever code. The goal is correct code that matches the spec. The spec is the product. Your job is to make it real.

---

## The Origin in One Sentence

A developer asked an AI to improve a Copilot cheat sheet. The AI produced three comprehensive guides. The developer asked: "if we know exactly what good configuration looks like, why make the developer write it?" Nine days later, the answer was a complete product specification for an invisible VS Code extension with seven autonomous workflows, eight file generators, a codebase dictionary, a learning database, and an MCP server - all designed to be built entirely by AI coding agents from documentation alone.

That's Roadie.
