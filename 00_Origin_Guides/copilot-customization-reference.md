# GitHub Copilot / VS Code AI Customization — Complete Reference
**Updated:** April 2026  
**Covers:** GitHub Copilot in VS Code (1.110+), Copilot CLI, Copilot Cloud Agent, JetBrains, Eclipse, Visual Studio 2026.  
**Spec:** Agent Skills follow the open [agentskills.io](https://agentskills.io) standard (30+ compatible tools).

---

## 1. `.github/` Folder Structure Overview

```text
your-repo/
├── AGENTS.md                          # Cross-agent always-on instructions (root)
├── CLAUDE.md                          # Claude Code always-on instructions (root)
├── GEMINI.md                          # Gemini always-on instructions (root)
└── .github/
    ├── copilot-instructions.md        # Copilot-specific always-on instructions
    ├── instructions/
    │   └── *.instructions.md          # Path-scoped targeted rules (applyTo globs)
    ├── prompts/
    │   └── *.prompt.md                # Reusable slash-command macros (/name)
    ├── agents/
    │   └── *.agent.md                 # Custom agent personas with tools & handoffs
    ├── skills/
    │   └── {skill-name}/
    │       ├── SKILL.md               # Required — skill definition (name must match dir)
    │       └── (optional resources)   # Scripts, templates, reference docs, examples
    ├── hooks/
    │   └── *.json                     # Agent lifecycle hooks (deterministic shell commands)
    └── workflows/
        ├── copilot-setup-steps.yml    # Reserved — cloud agent environment setup
        └── *.md                       # Agentic workflows (YAML frontmatter + Markdown body)
```

### Scope Comparison

| File/Folder | Always-on? | Scoped? | Tool Context | Where |
| :--- | :--- | :--- | :--- | :--- |
| `copilot-instructions.md` | ✅ Every request | Repo-wide | Copilot (all surfaces) | `.github/` |
| `AGENTS.md` | ✅ Every request | Repo-wide | All agents | Root (or any parent dir) |
| `CLAUDE.md` | ✅ Every request | Repo-wide | Claude Code | Root |
| `GEMINI.md` | ✅ Every request | Repo-wide | Gemini CLI | Root |
| `instructions/*.instructions.md` | ✅ Auto (via `applyTo`) | File/folder | Copilot, CLI | `.github/instructions/` |
| `prompts/*.prompt.md` | ❌ On-demand (`/name`) | Task | Copilot | `.github/prompts/` |
| `agents/*.agent.md` | ❌ When selected | Role | Copilot, CLI, Cloud | `.github/agents/` |
| `skills/{name}/SKILL.md` | ❌ When invoked/matched | Capability | Copilot, CLI, Cloud | `.github/skills/` |
| `hooks/*.json` | ✅ At lifecycle events | Event type | Copilot, CLI, Cloud | `.github/hooks/` |
| `workflows/*.md` | ❌ Trigger-based | Automation | Cloud Agent | `.github/workflows/` |

### Instruction Priority Order (highest → lowest)

1. **Personal instructions** — user-level (`~/.copilot/`, VS Code user profile)
2. **Repository instructions** — `.github/copilot-instructions.md` or `AGENTS.md`
3. **Organization-level instructions** — defined at GitHub org level
4. **File-based instructions** — `*.instructions.md` matched by `applyTo`

When conflicts exist, higher-priority instructions take precedence.

---

## 2. `.github/copilot-instructions.md`

**What it is:** The primary always-on instruction file. Automatically injected into every Copilot chat request in the repository. No frontmatter — pure Markdown.

**Supported in:** VS Code, github.com, Copilot CLI, Copilot Cloud Agent, Code Review, JetBrains, Eclipse, Xcode.

**Quick start:** Type `/init` in chat to auto-generate one tailored to your codebase. Or type `/create-instructions` followed by a description.

**Tips:**
- Keep it under ~500 lines to minimize context costs.
- Use structural headings — the agent parses them for relevance.
- Focus on rules relevant to *almost every* task. Use skills for specialized procedures.

```markdown
# Project Overview
Brief description of what this repo does and its primary purpose.

## Tech Stack
- Language: TypeScript 5.x (strict mode)
- Framework: Next.js 14 (App Router)
- Database: PostgreSQL via Prisma ORM
- Styling: Tailwind CSS v4
- Testing: Vitest + Playwright

## Coding Standards
- Use named exports only — no default exports
- All async functions must have try/catch with structured logging
- Prefer `const` over `let` unless mutation is required
- Explicit typing over implicit inference

## Architecture Patterns
- Feature-folder structure: /features/{name}/{components,hooks,utils}
- Server components by default; `"use client"` only for hooks/events
- Server state via TanStack Query — never fetch in `useEffect`

## Banned / Avoid
- Do NOT use `any` type in TypeScript
- Do NOT write raw SQL — use the Prisma client
- Do NOT use class components in React

## PR / Commit Standards
- Conventional commits: feat|fix|chore|docs|refactor|test
- PR title must reference an issue number

## Build & Run
- `pnpm install` → `pnpm dev` → http://localhost:3000
- Tests: `pnpm test` (unit), `pnpm test:e2e` (Playwright)
```

### Deprecation Notice

Settings-based code generation and test generation instructions (`github.copilot.chat.codeGeneration.instructions`, etc.) are **deprecated as of VS Code 1.102**. Use file-based instructions instead. Settings are still valid for code review, commit messages, and PR descriptions.

---

## 3. `.github/instructions/`

**What it is:** Path-scoped instruction files. Each file targets specific file types or folders via `applyTo` glob frontmatter. Loaded automatically when Copilot works on a matching file.

### Frontmatter

| Field | Required | Type | Description |
| :--- | :--- | :--- | :--- |
| `applyTo` | ✅ | glob string | Which files trigger this instruction. Supports comma-separated patterns. |

**Example: `react.instructions.md`**
```markdown
---
applyTo: "**/*.tsx,**/*.jsx"
---

## React Component Rules
- Functional components only — no class components
- Props interface named `{ComponentName}Props`, defined above the component
- Server components by default; add `"use client"` only when hooks/events needed

## State Management
- Local state: `useState`
- Server state: TanStack Query — never fetch in `useEffect`
```

**Example: `database.instructions.md`**
```markdown
---
applyTo: "**/prisma/**,**/db/**,**/*.prisma"
---

## Database Rules
- All schema changes go through Prisma migrations — never raw SQL
- Use transactions for multi-table writes
- Always include `@updatedAt` on mutable models
```

### Organization-Level Instructions

You can define instructions at the GitHub organization level so they apply across all repos in the org. Enable with the setting: `github.copilot.chat.organizationCustomAgents.enabled`. These appear in the Chat Instructions menu alongside personal and workspace instructions.

---

## 4. `.github/prompts/`

**What it is:** Reusable slash-command macros invoked on-demand via `/prompt-name` in chat.

**Quick start:** Type `/create-prompt` followed by a description to generate one from conversation.

### Frontmatter

| Field | Required | Type | Description |
| :--- | :--- | :--- | :--- |
| `description` | ✅ | string | Shown in prompt picker UI. |
| `mode` | ❌ | string | `ask` (chat), `edit` (inline diffs), `agent` (full agent mode). |
| `tools` | ❌ | string[] | Array of allowed tools (e.g., `codebase`, `create_file`, `web/fetch`). |

> **Note:** When tools are specified in both a custom agent and a prompt file, the prompt file's tools take precedence.

**Example: `security-audit.prompt.md`**
```markdown
---
description: Audit the current file for common security vulnerabilities
mode: ask
tools: [codebase, read_file, web/fetch]
---

Perform a security audit of the current file. Check for:
1. **Injection** — SQL injection, command injection, XSS
2. **Auth** — Missing authorization checks, insecure references
3. **Secrets** — Hardcoded credentials, API keys, tokens

For each finding: show the vulnerable line, explain the risk, and suggest a fix.
```

**Example: `create-component.prompt.md`**
```markdown
---
description: Scaffold a new React component with tests
mode: agent
tools: [create_file, codebase, terminal]
---

Create a new React component based on the user's description:
1. Create the component file in the appropriate feature folder
2. Create a co-located test file using `{name}.test.tsx`
3. Export from the feature's `index.ts` barrel file
4. Follow all conventions in the project's instructions
```

### Prompt vs Skill vs Agent — When to Use What

| Need | Use |
| :--- | :--- |
| One-off task automation (no tool restrictions needed) | **Prompt file** |
| Persistent persona with role + tool restrictions + handoffs | **Custom agent** |
| Portable, reusable capability with bundled scripts/templates | **Agent skill** |

---

## 5. `.github/agents/`

**What it is:** Custom agent profiles with specific tools, models, MCP servers, and handoff capabilities. When selected, the agent's instructions are prepended to every user prompt.

**Supported in:** VS Code, github.com (Cloud Agent), JetBrains, Eclipse, Xcode, Visual Studio 2026.

### Frontmatter

| Field | Required | Type | Description |
| :--- | :--- | :--- | :--- |
| `name` | ✅ | string | Unique identifier. |
| `description` | ✅ | string | What the agent does (shown in picker). |
| `tools` | ❌ | string[] | Whitelist of tools. Omit = all tools enabled. `[]` = no tools. |
| `model` | ❌ | string/array | Preferred model / fallback order (e.g., `['Claude Opus 4.5', 'GPT-5.2']`). |
| `handoffs` | ❌ | object[] | Transitions to other agents (`label`, `agent`, `prompt`, `send`). |
| `mcp-servers` | ❌ | object | MCP server configurations scoped to this agent. |
| `target` | ❌ | string | `vscode` or `github-copilot` to restrict where agent is available. Omit = both. |
| `hooks` | ❌ | object | Inline hook definitions (same format as `hooks/*.json`). |

> **Prompt limit:** The Markdown body can be a maximum of 30,000 characters.

**Example: `planner.agent.md`**
```markdown
---
name: Planner
description: Generates detailed implementation plans. Does NOT write code.
tools: [search/codebase, read_file, web/fetch, search/usages]
model: ['Claude Opus 4.5', 'GPT-5.2']
handoffs:
  - label: Implement This Plan
    agent: Implementer
    prompt: "Implement the plan outlined above. Follow all repo conventions."
    send: false
---

You are in **planning mode only**. Your sole job is to analyze a feature request and produce a detailed implementation plan.

## Your outputs must include:
1. **Scope** — which files will change and why
2. **Steps** — numbered, sequential implementation steps
3. **Risks** — potential breaking changes or edge cases
4. **Testing** — what tests to add or update

Do NOT edit any files or run any commands.
```

**Example: Agent with MCP server (`reviewer.agent.md`)**
```markdown
---
name: Reviewer
description: Reviews code against team standards using external style guide.
tools: [search/codebase, read_file, 'style-guide-mcp/get-rules']
mcp-servers:
  style-guide-mcp:
    type: local
    command: npx
    args: ['-y', 'style-guide-mcp-server']
    tools: ['*']
---

You are a code reviewer. Check all changes against:
1. Naming conventions from the style guide
2. Error handling patterns
3. Test coverage requirements
```

### Organization-Level Agents

Define agents at the GitHub org level to share across all repos. Enable with: `github.copilot.chat.organizationCustomAgents.enabled`.

---

## 6. `.github/skills/`

**What it is:** On-demand capability packages that load only when relevant (progressive disclosure). Invocable via `/skill-name` or automatically matched by the agent based on the description. An open standard compatible with 30+ tools (Copilot, Claude Code, Codex, Cursor, Gemini CLI, JetBrains Junie, etc.).

**Quick start:** Type `/create-skill` in chat to generate one, or extract from conversation: *"create a skill from how we just debugged that"*.

### Progressive Loading (3 levels)

1. **Discovery** — Only `name` + `description` from frontmatter are loaded into the system prompt.
2. **Instructions** — When matched, the full SKILL.md body is injected into context.
3. **Resources** — As the agent works through instructions, it accesses files in the skill directory only when referenced.

### `SKILL.md` Frontmatter

| Field | Required | Type | Description |
| :--- | :--- | :--- | :--- |
| `name` | ✅ | string | Lowercase, hyphens only. **Must match directory name.** Max 64 chars. |
| `description` | ✅ | string | Critical: Drives automatic discovery. Be specific about capabilities AND use cases. Max 1024 chars. |
| `allowed-tools` | ❌ | string[] | Pre-approved tools (e.g., `terminal`, `create_file`). |
| `argument-hint` | ❌ | string | Hint text shown when invoked as slash command (e.g., `[test file] [options]`). |
| `user-invocable` | ❌ | bool | Whether skill appears in `/` menu. Default: `true`. Set `false` for agent-only skills. |
| `disable-model-invocation` | ❌ | bool | Prevent automatic loading by the agent. Default: `false`. Set `true` for manual-only skills. |
| `license` | ❌ | string | License identifier (e.g., `MIT`). |

> **Security note:** Only pre-approve `terminal` / `shell` for highly trusted, self-authored skills.

**Example: `webapp-testing/SKILL.md`**
```markdown
---
name: webapp-testing
description: >-
  Guide for testing web applications using Playwright.
  Use when asked to create, fix, or run browser-based tests.
argument-hint: "File path or module name of the test target"
allowed-tools: [terminal]
license: MIT
---

# Web Application Testing with Playwright

## Procedure
1. Identify the subject file
2. Check for existing test file co-located (`{name}.test.ts`)
3. If none exists, create using [test-template.ts](./test-template.ts)
4. Cover: happy path, error path, boundary values
5. Run `pnpm test:e2e` and verify all pass

## Naming Convention
- Test files: `*.test.ts` with `describe/it` format
- Use AAA pattern (Arrange-Act-Assert)

## Reference
- See [example-scenarios.md](./example-scenarios.md) for common patterns
```

### Skill Directory Structure

```text
.github/skills/webapp-testing/
├── SKILL.md                    # Required — definition + instructions
├── test-template.ts            # Template the agent can use
├── example-scenarios.md        # Reference doc loaded on demand
└── scripts/
    └── run-coverage.sh         # Script the agent can execute
```

### Invocation Methods

| Method | Example |
| :--- | :--- |
| Slash command | `/webapp-testing for the login page` |
| Automatic match | *"help me test the login page"* → agent matches description |
| Type `/skills` | Opens the Configure Skills menu |

---

## 7. `.github/hooks/`

**What it is:** JSON configuration files that execute shell commands at specific agent lifecycle points. Unlike instructions or prompts that guide behavior through natural language, hooks run your code with **deterministic, guaranteed outcomes**.

**Quick start:** Type `/hooks` in chat to configure a new hook interactively.

**Status:** Preview (as of VS Code 1.110). Format may change.

### Hook Events

#### VS Code (8 events)

| Event | When It Fires | Common Use Cases |
| :--- | :--- | :--- |
| `SessionStart` | First prompt of a new session | Initialize resources, log session start, validate project state |
| `UserPromptSubmit` | User submits a prompt | Audit requests, inject system context |
| `PreToolUse` | Before agent invokes any tool | Block dangerous operations, require approval, modify input |
| `PostToolUse` | After tool completes successfully | Run formatters, log results, trigger follow-up actions |
| `PreCompact` | Before conversation context is compacted | Export important context, save state before truncation |
| `SubagentStart` | Subagent is spawned | Track nested agent usage, initialize subagent resources |
| `SubagentStop` | Subagent completes | Aggregate results, cleanup subagent resources |
| `Stop` | Session ends | Cleanup, generate reports, send notifications |

#### Copilot Cloud Agent / CLI (6 events)

| Event | When It Fires |
| :--- | :--- |
| `sessionStart` | New session begins or resumes |
| `sessionEnd` | Session completes or is terminated |
| `userPromptSubmitted` | User submits a prompt |
| `preToolUse` | Before tool invocation (can approve/deny) |
| `postToolUse` | After tool completes |
| `errorOccurred` | When an error occurs during execution |

> **Naming:** VS Code uses PascalCase (`PreToolUse`), CLI/Cloud use camelCase (`preToolUse`). VS Code auto-converts between formats for cross-tool compatibility.

### Hook File Locations (searched in order)

| Location | Scope |
| :--- | :--- |
| `.github/hooks/*.json` | Workspace — shared with team via version control |
| `.claude/settings.local.json` | Local workspace — not committed |
| `.claude/settings.json` | Workspace-level (Claude Code compat) |
| `~/.claude/settings.json` | User — personal hooks across all workspaces |
| `~/.copilot/hooks/` | User — personal hooks (Copilot-specific) |

Workspace hooks take precedence over user hooks for the same event type.

### Configuration Format

```json
{
  "version": 1,
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "./scripts/security-check.sh",
        "timeout": 15
      }
    ],
    "PostToolUse": [
      {
        "type": "command",
        "command": "npx prettier --write \"$TOOL_INPUT_FILE_PATH\"",
        "windows": "npx prettier --write \"%TOOL_INPUT_FILE_PATH%\""
      }
    ],
    "SessionStart": [
      {
        "type": "command",
        "command": "echo \"Session started\" >> .copilot/session.log"
      }
    ]
  }
}
```

### Hook Entry Properties

| Property | Required | Description |
| :--- | :--- | :--- |
| `type` | ✅ | Must be `"command"`. |
| `command` | ✅* | Shell command to run (default/linux/macOS). |
| `windows` | ❌ | Windows-specific command. |
| `bash` | ❌ | Maps to `osx` + `linux` (CLI format). |
| `powershell` | ❌ | Maps to `windows` (CLI format). |
| `timeout` | ❌ | Seconds before timeout. Default: 30. |
| `cwd` | ❌ | Working directory for the command. |
| `env` | ❌ | Environment variables object. |

> *At least one of `command`, `bash`, or `powershell` is required.

### Inline Hooks in Agent Definitions

You can embed hooks directly in a custom agent's frontmatter:

```markdown
---
name: Strict Formatter
description: Agent that auto-formats code after every edit
hooks:
  PostToolUse:
    - type: command
      command: "./scripts/format-changed-files.sh"
---

You are a code editing agent. After making changes, files are automatically formatted.
```

### PreToolUse Hook Output (Approval Control)

PreToolUse hooks can control whether the tool executes by outputting JSON to stdout:

```json
{ "decision": "deny", "reason": "Blocked: rm -rf not allowed" }
```

Valid decisions: `allow`, `deny`, `ask`. When multiple hooks target the same event, the most restrictive wins (`deny` > `ask` > `allow`).

### Security Considerations

- Hooks execute with the same permissions as VS Code / the CLI.
- **Always review** hook scripts from shared repositories before enabling.
- Never hardcode secrets — use environment variables.
- Validate and sanitize all JSON input from stdin.

---

## 8. `.github/workflows/`

**What it is:** GitHub Actions directory with a reserved setup file for the cloud agent and/or natural language agentic workflows.

### Reserved File: `copilot-setup-steps.yml`

Provisions the cloud agent's environment so it doesn't waste time guessing configurations. **Must** use the job name `copilot-setup-steps`.

```yaml
name: "Copilot Setup Steps"
on: workflow_dispatch

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
```

### Agentic Workflows (`*.md` with YAML frontmatter)

Natural language workflows that combine GitHub Actions triggers with AI-driven execution. The `gh aw compile` command converts `.md` files into hardened `.lock.yml` GitHub Actions workflows.

> **Note:** Agentic workflow files use the `.md` extension (not `.yml`). The compiled lock file is `.lock.yml`.

**Frontmatter Fields:**

| Field | Description |
| :--- | :--- |
| `on` | GitHub Actions trigger (issues, schedule, pull_request, etc.) |
| `permissions` | GitHub token permissions (`read-all`, `contents: read`, etc.) |
| `safe-outputs` | Constrained output actions (e.g., `create-issue`, `add-comment`) |
| `engine` | `copilot` (default) |
| `timeout-minutes` | Max execution time |

**Example: `weekly-dependency-report.md`**
```markdown
---
on:
  schedule:
    - cron: "0 9 * * 1"
permissions:
  contents: read
  issues: write
safe-outputs:
  create-issue:
    title-prefix: "[deps] "
    labels: [dependencies, automated]
---

# Weekly Dependency Report

## Task
Analyze the repository's dependencies and create a summary issue.

## Steps
1. Read `package.json` and `pnpm-lock.yaml`
2. Check for outdated packages
3. Group findings by severity (critical, high, moderate, low)
4. Create a GitHub issue with a summary table of findings
```

---

## 9. Agent Plugins (Preview)

**What it is:** Prepackaged bundles of customizations (skills, agents, hooks, MCP servers, slash commands) that you can discover and install from plugin marketplaces.

**Where:** Extensions view in VS Code → Agent Plugins section.

**Marketplaces:** By default, VS Code discovers plugins from `copilot-plugins` and `awesome-copilot` repositories. Add additional marketplaces via `chat.plugins.marketplaces` setting.

### What a Plugin Can Bundle

- Slash commands
- Agent skills (with bundled resources)
- Custom agents
- Hooks
- MCP servers (auto-start when plugin enabled, auto-stop when disabled)

### Plugin Structure

```text
my-plugin/
├── plugin.json         # Plugin manifest (agents, commands, skills)
├── .mcp.json           # Optional — MCP server definitions
├── skills/
│   └── my-skill/
│       └── SKILL.md
├── agents/
│   └── my-agent.agent.md
└── hooks/
    └── my-hooks.json
```

### Managing Plugins

- **Install:** Browse in Extensions view → Agent Plugins marketplace
- **Disable:** Right-click → Disable (all bundled customizations stop)
- **Uninstall:** Right-click → Uninstall (external sources removed from disk)
- Plugin hooks run alongside workspace and user hooks. For PreToolUse, the most restrictive decision across all hooks wins.

---

## 10. MCP Servers

**What it is:** The Model Context Protocol (MCP) is an open standard for connecting AI models to external data sources, APIs, and tools. MCP extends agent capabilities beyond the codebase.

### Configuration Locations

| Location | Scope |
| :--- | :--- |
| `.vscode/mcp.json` | VS Code workspace |
| Agent frontmatter (`mcp-servers:`) | Scoped to a specific agent |
| Repository settings on github.com | Cloud Agent (repo-level) |
| Plugin `.mcp.json` | Bundled with a plugin |

### Cloud Agent MCP Config (Repository Settings)

```json
{
  "mcpServers": {
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.dev/sse",
      "tools": ["search_issues", "get_issue_details"],
      "env": {
        "SENTRY_AUTH": "$COPILOT_MCP_SENTRY_TOKEN"
      }
    }
  }
}
```

> **Security:** Cloud Agent uses MCP tools autonomously without asking for approval. Always allowlist specific read-only tools rather than using `"*"`. Secrets must be prefixed with `COPILOT_MCP_` and stored in the Copilot environment.

### Agent-Scoped MCP (in frontmatter)

```yaml
mcp-servers:
  custom-mcp:
    type: local
    command: some-command
    args: ['--arg1']
    tools: ['tool-1', 'tool-2']
    env:
      API_KEY: ${{ secrets.COPILOT_MCP_API_KEY }}
```

Supported types: `local`, `stdio`, `http`, `sse`. The `stdio` type (Claude Code / VS Code) is automatically mapped to `local` for Cloud Agent compatibility.

---

## 11. Root-Level Files (Multi-Agent Support)

Root files standardize behavior across multiple AI assistants. VS Code discovers these by walking up from the workspace folder to the repository root (enable `chat.useCustomizationsInParentRepositories` for monorepos).

| File | Prioritized By | Notes |
| :--- | :--- | :--- |
| `AGENTS.md` | All agents (open standard) | Cross-tool always-on instructions |
| `CLAUDE.md` | Claude Code | Claude-specific instructions |
| `GEMINI.md` | Gemini CLI | Gemini-specific instructions |
| `.github/copilot-instructions.md` | Copilot only | Copilot-specific instructions |

**Example: `AGENTS.md`**
```markdown
# Project: Questerix

## Overview
Educational technology platform with Flutter student app, React admin panel, and Supabase backend.

## Tech Stack
- Frontend: React 19, TypeScript 5.x, TanStack Query, shadcn/ui
- Mobile: Flutter (Riverpod, Drift)
- Backend: Supabase (Edge Functions, Row-Level Security)
- Infra: Cloudflare (Workers, Pages, D1, R2)

## Coding Standards
- TypeScript strict mode — no `any`, no implicit returns
- Named exports only
- Tests required for all public APIs
- Prefer composition over inheritance

## Out of Bounds
- NEVER commit secrets or API keys
- NEVER modify existing migration files
- NEVER use `console.log` in production code — use the structured logger
```

### Monorepo Discovery

For monorepos where you open a subfolder as your workspace:

```text
my-monorepo/          # repo root (has .git)
├── .github/
│   ├── copilot-instructions.md    # ← discovered
│   └── agents/
│       └── reviewer.agent.md      # ← discovered
├── packages/
│   └── frontend/     # ← opened as workspace folder
│       └── .github/
│           └── instructions/
│               └── react.instructions.md  # ← also discovered
```

Enable: `chat.useCustomizationsInParentRepositories` in VS Code settings.

---

## 12. Personal / Home Directory Files

Apply to *all* projects on your machine, not just specific repositories.

### Supported Locations

| Path | Tool |
| :--- | :--- |
| `~/.copilot/copilot-instructions.md` | Copilot (always-on) |
| `~/.copilot/skills/{name}/SKILL.md` | Copilot (personal skills) |
| `~/.copilot/hooks/` | Copilot (personal hooks) |
| `~/.claude/skills/{name}/SKILL.md` | Claude Code (personal skills) |
| `~/.claude/settings.json` | Claude Code (settings + hooks) |
| `~/.agents/skills/{name}/SKILL.md` | Generic agent-compatible skills |

> Personal skills are not committed to any repository, making them ideal for personal preferences or environment-specific procedures.

**Sync across machines:** Enable Settings Sync → run `Settings Sync: Configure` → select *Prompts and Instructions*.

**Example: `~/.copilot/copilot-instructions.md`**
```markdown
# Developer Personal Instructions

## My Defaults
- I prefer explicit typing over implicit inference
- Always use `const` over `let` unless mutation is required
- Short functions (< 30 lines) over long ones
- Comments explain *why*, not *what*

## Environment
- Shell: zsh
- Package manager: pnpm
- Editor: VS Code with Vim keybindings

## Communication Style
- Be direct and concise
- Skip preamble — get to the code
- When reviewing my code, be ruthlessly honest
```

---

## 13. Useful Chat Commands

| Command | What It Does |
| :--- | :--- |
| `/init` | Generate `copilot-instructions.md` tailored to your codebase |
| `/create-instructions` | Generate instructions from a description |
| `/create-prompt` | Generate a prompt file from a description |
| `/create-skill` | Generate a skill from a description or extract from conversation |
| `/hooks` | Configure a new hook interactively |
| `/skills` | Open the Configure Skills menu |
| `/compact` | Manually compact conversation history (e.g., `/compact forget all variants except Rust`) |
| `/autoApprove` or `/yolo` | Toggle global auto-approval of tool calls |
| `Chat: Open Chat Customizations` | Open the unified customization editor (Command Palette) |

---

## 14. Quick Decision Guide

```text
What do you need?
│
├── Apply to EVERY request, always
│   ├── All agents (Copilot + Claude + Gemini + others) → AGENTS.md (root)
│   ├── Copilot only → .github/copilot-instructions.md
│   ├── Claude Code only → CLAUDE.md (root)
│   └── Personal / across all repos → ~/.copilot/copilot-instructions.md
│
├── Apply only to specific file types or folders
│   └── .github/instructions/*.instructions.md  (use applyTo glob)
│
├── Reusable slash-command macro (one-off tasks)
│   └── .github/prompts/*.prompt.md  (invoke with /name)
│
├── Persistent agent persona with role + tool restrictions + handoffs
│   └── .github/agents/*.agent.md  (select from agent picker)
│
├── Repeatable workflow with scripts/templates the agent can call
│   └── .github/skills/{name}/SKILL.md  (loads only when relevant)
│
├── Extend agent with external data/APIs
│   └── MCP servers (.vscode/mcp.json, agent frontmatter, or repo settings)
│
├── Prepackaged bundle of skills + agents + hooks + MCP
│   └── Agent plugins (install from Extensions view)
│
├── Run code at agent lifecycle events (format, validate, block, audit)
│   └── .github/hooks/*.json  (deterministic shell commands)
│
└── Automated unattended agent task (scheduled, event-triggered)
    ├── Provision agent's environment → .github/workflows/copilot-setup-steps.yml
    └── Natural language automation → .github/workflows/*.md (compiled to .lock.yml)
```

---

## 15. Cross-Tool Compatibility Matrix

The Agent Skills spec (`agentskills.io`) and hook formats are designed for portability. Here's what works where:

| Feature | VS Code | Copilot CLI | Cloud Agent | Claude Code | Gemini CLI | JetBrains |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `copilot-instructions.md` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `AGENTS.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `CLAUDE.md` | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `*.instructions.md` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `*.prompt.md` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `*.agent.md` | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ |
| `SKILL.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅* |
| Hooks (`.json`) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| MCP servers | ✅ | Partial | ✅ | ✅ | ✅ | ✅ |
| Agent plugins | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

*\* JetBrains Junie supports skills via the open standard.*

---

## 16. Key Resources

- **Official Docs:** [code.visualstudio.com/docs/copilot/customization](https://code.visualstudio.com/docs/copilot/customization/overview)
- **GitHub Docs:** [docs.github.com/copilot](https://docs.github.com/en/copilot)
- **Community Collection:** [github.com/github/awesome-copilot](https://github.com/github/awesome-copilot)
- **Agent Skills Spec:** [agentskills.io](https://agentskills.io)
- **Customization Handbook:** [copilot-academy.github.io/workshops/copilot-customization](https://copilot-academy.github.io/workshops/copilot-customization/copilot_customization_handbook)
- **Anthropic Skills Repo:** [github.com/anthropics/skills](https://github.com/anthropics/skills)
