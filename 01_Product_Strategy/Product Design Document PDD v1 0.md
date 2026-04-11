# Product Design Document (PDD) — v1.0

**CONFIDENTIAL** — Version 1.0 — April 2026

---

# Product Vision

Roadie is an invisible, autonomous VS Code extension that transforms GitHub Copilot from a reactive code-completion tool into a proactive, workflow-driven engineering partner. It sits between the developer and Copilot as a transparent orchestration layer: intercepting natural language intent from chat, classifying tasks into structured workflows, and executing sophisticated multi-step operations—bug fixing, feature development, refactoring, code review, documentation, dependency management—while the developer experiences nothing more than an unusually capable chat interface. Simultaneously, Roadie maintains a persistent internal model of the project by watching the file system and analyzing code structure, using this model to silently generate and maintain AI configuration files (.github/[copilot-instructions.md](http://copilot-instructions.md), custom agents, skills, hooks, workflows) so that every Copilot interaction—whether through Roadie or not—is precisely tuned to the project’s actual technology stack, coding patterns, and architectural decisions. The developer installs the extension and encounters no setup wizard, no configuration screen, no onboarding flow. The first time they chat with Copilot after installation, responses are noticeably more accurate, context-aware, and complete. That moment of quiet surprise—“wait, how did it know that?”—is the product’s first impression and its enduring promise: intelligence that compounds over time, works entirely on the developer’s machine, requires zero configuration, and makes every AI tool that touches the repository smarter by generating portable, standards-based configuration files that travel with the code.

---

# User Experience Narrative

## Day Zero: Installation

Maya is a solo developer building a SaaS application with Next.js, Prisma, and PostgreSQL. She installs Roadie from the VS Code Marketplace. There is no activation prompt, no welcome tab, no configuration wizard. The extension activates silently. A single, brief notification appears in the status bar: “Roadie active.” It fades. Nothing else changes.

## First Interaction: The Quiet Difference

Maya opens the Copilot chat panel and selects “Roadie” from the agent dropdown—the same place she’d pick @workspace or @terminal. She types: “The user profile page is throwing a 500 error after the last deploy.” She expects what she always gets from Copilot: a generic suggestion to check her error logs. Instead, Roadie’s intent classifier identifies this as a bug-fix trigger. The bug-fix workflow activates. In the chat, she sees progress indicators—“Locating error source...”, “Analyzing stack trace...”, “Generating fix...”—and then a complete response: the root cause (a null reference in the profile serializer introduced by a recent migration), a code fix applied to the correct file, a verification step that ran her test suite (which passed), a scan for similar patterns elsewhere in the codebase (two more found, both fixed), and a regression test added to prevent recurrence. The entire exchange took ninety seconds. Maya did not navigate to a single file, did not read a stack trace, did not run a test manually. She typed one sentence and got a complete resolution.

## The Background Awakening

While Maya worked, Roadie’s project analyzer was lazily building its internal model. Not all at once—it gathered what the bug-fix workflow needed: Next.js 14 with App Router, Prisma ORM with PostgreSQL, Vitest for testing, pnpm as the package manager. It wrote this context into .github/[copilot-instructions.md](http://copilot-instructions.md), noting the tech stack, the testing conventions, and the project structure. It generated a .github/agents/[debugger.agent.md](http://debugger.agent.md) file tuned to this stack. These files appeared as unstaged changes in Maya’s Source Control view. She glanced at them, saw they were reasonable, and committed them. She didn’t have to.

## Daily Use: The Invisible Partner

Over the next week, Maya uses Roadie without thinking about it. She types “Add a dark mode toggle to the settings page” and gets a feature-development workflow: Roadie presents a plan (database flag in user preferences, API endpoint, React component with system-preference detection, CSS variable theming), waits for her approval, then executes all layers in parallel—she sees progress for each track and the consolidated result. She types “Refactor the authentication module—it’s gotten messy” and gets a refactoring workflow that writes characterization tests first, refactors incrementally, and verifies tests pass after each step. She types “Review my changes before I push” and gets a multi-perspective code review covering security, performance, code quality, and test coverage.

Each time a workflow runs, the project model deepens. Roadie learns that Maya uses barrel exports, prefers early returns, follows a specific commit message convention. It updates the generated instruction files to reflect these patterns. When Maya makes a change to a generated file—editing a rule she disagrees with, adding a convention Roadie missed—the edit is preserved on subsequent regenerations. The tool adapts to her; she never adapts to it.

## The Compounding Effect

After a month, Maya’s .github/ directory contains a rich set of AI configuration files that encode her project’s intelligence. When she uses Copilot’s inline completions (not through Roadie), they’re better—because Copilot reads [copilot-instructions.md](http://copilot-instructions.md). When she tries Claude Code on a branch, it reads [AGENTS.md](http://AGENTS.md) and understands the project’s conventions immediately. The intelligence Roadie generated is portable. It outlives the extension. If Maya uninstalls Roadie tomorrow, the generated files remain, and every AI tool that touches her repository still benefits from them.

[PDD — Agent Architecture, Project Model & File Generation](Product%20Design%20Document%20(PDD)%20%E2%80%94%20v1%200/PDD%20%E2%80%94%20Agent%20Architecture,%20Project%20Model%20&%20File%20Gen%2033fc821ae63c811a8920c70f3976a85e.md)

[PDD — Configuration, Privacy, Scope & Metrics](Product%20Design%20Document%20(PDD)%20%E2%80%94%20v1%200/PDD%20%E2%80%94%20Configuration,%20Privacy,%20Scope%20&%20Metrics%2033fc821ae63c816dbd7fea800c41b69c.md)