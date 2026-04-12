# GitHub Copilot Agents — Creation, Orchestration & Delegation Guide
**Updated:** April 2026  
**Covers:** VS Code 1.110+, Copilot CLI, Copilot Cloud Agent  
**Prerequisite:** Familiarity with `.github/agents/` folder structure (see the Customization Reference).

---

## 1. Core Concepts

### What Is a Custom Agent?

A custom agent is a `.agent.md` file that **replaces the model's identity** — not just its instructions. When you switch to a custom agent, you're changing:

- **Who** the model is (persona, role, expertise)
- **What** it can do (tool whitelist)
- **Which model** powers it (can differ per agent)
- **Where** it can hand off to next (handoffs, subagent delegation)

This is fundamentally different from instructions (which add rules) or skills (which add capabilities). Agents change the model's entire operating context.

### The Three Workflow Patterns

| Pattern | Control | Context | Best For |
| :--- | :--- | :--- | :--- |
| **Handoffs** | User-controlled (button click) | Transfers — conversation moves to target agent | Sequential workflows with review gates |
| **Subagents** | Agent-controlled (autonomous) | Isolated — subagent gets fresh context, returns summary only | Focused subtasks, parallel research, exploration |
| **Delegation** | User-initiated → autonomous | Transfers to a different runtime | Cross-environment work (local → CLI → cloud) |

### The Three Invocation Patterns

| Method | How | When |
| :--- | :--- | :--- |
| **Direct selection** | User picks agent from dropdown | Interactive sessions |
| **`agents` property** | Frontmatter whitelists subagent names | Orchestrator delegates to workers |
| **`runSubagent` tool** | Model invokes tool autonomously | Agent decides delegation is beneficial |

---

## 2. Agent Anatomy — Full Frontmatter Reference

```yaml
---
# === Identity ===
name: string                    # Unique identifier (defaults to filename without .agent.md)
description: string             # Required. What the agent does. Shown in picker.
argument-hint: string           # Optional. Placeholder text in chat input.

# === Capabilities ===
tools: string[]                 # Tool whitelist. Omit = all tools. [] = no tools.
                                # Include 'agent' to enable subagent delegation.
                                # Use '<server>/*' to include all tools from an MCP server.
agents: string[]                # Subagent whitelist. '*' = all agents. [] = no subagents.
                                # Names must match other .agent.md `name` fields.
model: string | string[]        # Model or fallback list: ['Claude Opus 4.5', 'GPT-5.2']
                                # Format: 'Model Name (vendor)' e.g. 'GPT-5 (copilot)'

# === MCP Servers (scoped to this agent) ===
mcp-servers:                    # Optional. Agent-scoped MCP server configs.
  server-name:
    type: local | stdio | http | sse
    command: string
    args: string[]
    tools: string[]             # Allowlist specific tools or use ['*']
    env:
      KEY: ${{ secrets.COPILOT_MCP_KEY }}

# === Workflow ===
handoffs:                       # Optional. Sequential workflow transitions.
  - label: string               # Button text shown to user
    agent: string               # Target agent name
    prompt: string              # Pre-filled prompt for the target
    send: bool                  # true = auto-submit, false = user reviews first
    model: string               # Optional. Override model for the handoff.

hooks:                          # Optional. Inline lifecycle hooks.
  PostToolUse:
    - type: command
      command: string

# === Visibility ===
target: vscode | github-copilot # Optional. Restrict to one environment. Omit = both.
user-invocable: bool            # Default: true. false = hidden from picker (subagent-only).
disable-model-invocation: bool  # Default: false. true = can't be auto-invoked as subagent.
---

# Markdown body — the agent's prompt (max 30,000 characters)
```

### Visibility Matrix

| `user-invocable` | `disable-model-invocation` | In Picker? | As Subagent? |
| :--- | :--- | :--- | :--- |
| `true` (default) | `false` (default) | ✅ | ✅ |
| `true` | `true` | ✅ | ❌ |
| `false` | `false` | ❌ | ✅ |
| `false` | `true` | ❌ | ❌ (dead agent) |

---

## 3. Building Agents — Practical Examples

### 3.1 Read-Only Planner

A planning agent that **cannot** edit files. Designed to produce implementation plans, then hand off.

```markdown
---
name: Planner
description: Generates detailed implementation plans. Does NOT write code.
tools: ['search/codebase', 'search/usages', 'read_file', 'web/fetch']
model: ['Claude Opus 4.5', 'GPT-5.2']
handoffs:
  - label: Implement This Plan
    agent: Implementer
    prompt: "Implement the plan outlined above. Follow all repo conventions."
    send: false
  - label: Write Tests First (TDD)
    agent: TDD-Writer
    prompt: "Write failing tests for the plan above. Do not implement yet."
    send: false
---

# Planning Mode

You are a **planning-only** agent. You analyze feature requests and produce
detailed implementation plans. You NEVER edit files or run commands.

## Your outputs must include:
1. **Scope** — which files will change and why
2. **Steps** — numbered, sequential implementation steps with file paths
3. **Dependencies** — new packages, API changes, or migrations needed
4. **Risks** — potential breaking changes, edge cases, performance concerns
5. **Testing strategy** — what tests to add or update

## Rules
- Reference specific file paths and function names from the codebase
- If the request is ambiguous, ask clarifying questions BEFORE planning
- Keep plans actionable — another agent will implement them verbatim
```

### 3.2 Focused Implementer

Pairs with the Planner above. Has full edit access but no web/research tools.

```markdown
---
name: Implementer
description: Implements plans produced by the Planner agent.
tools: ['edit_file', 'create_file', 'delete_file', 'terminal',
        'search/codebase', 'read_file']
model: 'Claude Sonnet 4.5'
handoffs:
  - label: Review Changes
    agent: Reviewer
    prompt: "Review all changes made in this session for quality and correctness."
    send: false
---

# Implementation Mode

You implement plans exactly as specified. Follow the plan step-by-step.

## Rules
- Do NOT deviate from the plan without explicit approval
- After each file change, verify it compiles/lints: run the appropriate check
- If a step is unclear, STOP and explain what's ambiguous — do not guess
- Write concise commit messages for each logical change
```

### 3.3 Security Reviewer (Subagent-Only)

Hidden from the picker — only invocable as a subagent by an orchestrator.

```markdown
---
name: SecurityReviewer
description: Scans code for security vulnerabilities using OWASP guidelines.
user-invocable: false
tools: ['search/codebase', 'read_file', 'terminal']
model: 'Claude Opus 4.5'
---

# Security Review Agent

You are a security specialist. Analyze code for vulnerabilities.

## Check for:
1. **Injection** — SQL, command, XSS, template injection
2. **Authentication** — missing auth checks, insecure session handling
3. **Secrets** — hardcoded credentials, API keys, tokens in source
4. **Dependencies** — known CVEs in packages (run `npm audit` or equivalent)
5. **Data exposure** — PII in logs, overly permissive API responses

## Output format:
For each finding:
- **Severity**: Critical / High / Medium / Low
- **Location**: file:line
- **Description**: what's wrong
- **Recommendation**: specific fix
```

### 3.4 Documentation Agent with MCP

Connects to an external docs server for style guide enforcement.

```markdown
---
name: DocWriter
description: Writes and updates project documentation following the style guide.
tools: ['read_file', 'edit_file', 'create_file', 'search/codebase',
        'docs-mcp/get-style-rules', 'docs-mcp/validate-markdown']
mcp-servers:
  docs-mcp:
    type: http
    url: https://docs-mcp.internal.company.com/sse
    tools: ['get-style-rules', 'validate-markdown']
handoffs:
  - label: Back to Coding
    agent: agent
    prompt: "Documentation is updated. Continue with implementation."
    send: false
---

# Documentation Agent

You write clear, accurate technical documentation.

## Rules
- Always fetch style rules from the docs MCP before writing
- Use the validation tool to check all markdown before finishing
- Update the table of contents when adding new sections
- Cross-reference related docs with relative links
```

---

## 4. Handoffs — Sequential Workflows with User Control

Handoffs transfer the conversation from one agent to another. The user sees a button after the agent's response and decides when to proceed. This creates **review gates** between workflow phases.

### How Handoffs Work

```
┌──────────┐   user clicks    ┌──────────────┐   user clicks    ┌──────────┐
│ Planner  │ ──"Implement"──▶ │ Implementer  │ ──"Review"─────▶ │ Reviewer │
│ (plan)   │   button         │ (code)       │   button         │ (review) │
└──────────┘                  └──────────────┘                  └──────────┘
```

1. Agent A completes its response.
2. Handoff buttons appear below the response.
3. User clicks a button → VS Code switches to Agent B.
4. If `send: false` — the prompt appears in the input for user to review/edit.
5. If `send: true` — the prompt auto-submits immediately.

### Handoff Frontmatter

```yaml
handoffs:
  - label: "Start Implementation"     # Button text
    agent: Implementer                 # Target agent name (must match a .agent.md)
    prompt: "Implement the plan above" # Context passed to the target
    send: false                        # false = user reviews, true = auto-send
    model: "Claude Sonnet 4.5"         # Optional model override for the handoff
```

### Common Handoff Chains

**Plan → Implement → Review:**
```
Planner ──▶ Implementer ──▶ Reviewer ──▶ (done)
```

**TDD Workflow:**
```
Planner ──▶ TDD-Writer (failing tests) ──▶ Implementer (make tests pass) ──▶ Reviewer
```

**Feature + Docs:**
```
Implementer ──▶ DocWriter ──▶ Reviewer
```

### Design Tips

- Use `send: false` for transitions that need human judgment (plan → implement).
- Use `send: true` for mechanical transitions (lint → format → commit).
- Keep handoff prompts specific — include what the previous agent produced.
- Each agent in the chain should be independently useful (no circular dependencies).

---

## 5. Subagents — Agent-Controlled Delegation

Subagents are **context-isolated** workers that the main agent spawns autonomously. They get a fresh context window, do focused work, and return only the final result. The intermediate exploration stays contained.

### Key Properties

- **Synchronous** — main agent waits for results (but multiple subagents can run in parallel).
- **Context-isolated** — subagent gets only the task prompt, not the main conversation history.
- **Result-only return** — only the summary flows back to the main agent's context.
- **No nesting** — a subagent cannot invoke another subagent.
- **Tool inheritance** — by default uses same model/tools as main session, unless a custom agent overrides.

### Enabling Subagents

The `agent` tool must be in the agent's tools list, and the `agents` property controls which agents can be used as subagents:

```yaml
---
name: Orchestrator
tools: ['agent', 'edit', 'search', 'read', 'terminal']
agents: ['Planner', 'Implementer', 'SecurityReviewer', 'Reviewer']
---
```

| `agents` value | Behavior |
| :--- | :--- |
| `['Planner', 'Reviewer']` | Only these agents available as subagents |
| `['*']` | All agents available as subagents |
| `[]` | No subagent delegation allowed |
| *(omitted)* | All agents available (same as `*`) |

### How Subagent Execution Appears

In the chat, subagent work appears as a collapsible tool call:

```
▶ Subagent: SecurityReviewer — "Scanning for vulnerabilities..."
  ├── Reading file: src/auth/login.ts
  ├── Reading file: src/auth/middleware.ts
  ├── Running terminal: npm audit
  └── Result: 3 findings (1 High, 2 Medium)
```

Click to expand and see all tool calls, the prompt passed, and the full result.

### Invoking Subagents

**From a prompt file:**
```markdown
---
description: Document a feature with research
tools: ['agent', 'read', 'search', 'edit']
---

Run a subagent to research the feature implementation details.
Return only information relevant for user documentation.
Then update the docs/ folder with the new documentation.
```

**From user chat (hinting):**
```
Analyze this codebase for refactoring opportunities. Use subagents to:
1. Find duplicate code patterns
2. Identify unused exports
3. Check for security vulnerabilities

Compile findings into a prioritized action plan.
```

**From agent instructions (embedded in the orchestrator prompt):**
```markdown
For each feature request:
1. Delegate to the Planner subagent to break down the feature into tasks.
2. Delegate to the SecurityReviewer subagent to check the affected areas.
3. Synthesize their findings into a final implementation brief.
```

### Creating Subagent-Only Agents

Agents that should never appear in the picker — only available for delegation:

```yaml
---
name: InternalResearcher
description: Researches codebase patterns and returns analysis.
user-invocable: false          # Hidden from picker
tools: ['search/codebase', 'read_file', 'web/fetch']
model: 'Claude Haiku 4.5'     # Fast, cheap model for research
---
```

### Self-Referencing Agents (Recursive Delegation)

An agent can delegate to itself for divide-and-conquer patterns. Requires the setting: `chat.subagents.allowInvocationsFromSubagents`.

```markdown
---
name: RecursiveProcessor
tools: ['agent', 'read', 'search']
agents: [RecursiveProcessor]
argument-hint: A list of items to process
---

You process a list of items by dividing and conquering:
- If the list has more than 4 items, split it in half and delegate each half
  to a RecursiveProcessor subagent.
- If the list has 4 or fewer items, process them directly.
- Merge the results from each subagent into a final result.
```

---

## 6. The Orchestrator Pattern

The orchestrator pattern combines handoffs and subagents into a coordinated system where a **central coordinator** decomposes complex tasks and delegates to specialists.

### Architecture

```
                    ┌─────────────────────┐
                    │    Orchestrator      │
                    │  (coordinator agent) │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │   Planner   │ │ Implementer │ │  Reviewer    │
    │ (read-only) │ │ (full edit) │ │ (read-only)  │
    └─────────────┘ └─────────────┘ └─────────────┘
              │               │               │
              ▼               ▼               ▼
         [plan.md]      [code changes]   [review.md]
```

### Full Orchestrator Example

**`.github/agents/feature-builder.agent.md`**
```markdown
---
name: FeatureBuilder
description: >-
  Orchestrates end-to-end feature development. Delegates planning, architecture
  validation, implementation, and review to specialized subagents.
tools: ['agent', 'edit', 'search', 'read', 'terminal']
agents: ['Planner', 'Architect', 'Implementer', 'SecurityReviewer', 'Reviewer']
model: 'Claude Opus 4.5'
handoffs:
  - label: Deploy & Monitor
    agent: Deployer
    prompt: "All changes are reviewed and approved. Proceed with deployment."
    send: false
---

# Feature Builder — Orchestrator

You are the coordinator for end-to-end feature development. You NEVER write
code yourself. You delegate all work to specialized subagents and synthesize
their outputs.

## Workflow

### Phase 1: Planning
1. Delegate to **Planner** — produce an implementation plan
2. Delegate to **Architect** — validate the plan against codebase patterns
3. If the Architect identifies issues, send feedback to a new Planner subagent
   to revise the plan
4. Present the final plan to the user for approval before proceeding

### Phase 2: Implementation
5. Delegate to **Implementer** — execute the approved plan step by step
6. After implementation, delegate to **SecurityReviewer** — scan all changed
   files for vulnerabilities

### Phase 3: Review
7. Delegate to **Reviewer** — check code quality, test coverage, and adherence
   to the plan
8. If the Reviewer flags issues, delegate back to **Implementer** with specific
   fix requests
9. Present a final summary to the user

## Rules
- Always present the plan to the user before Phase 2
- Run SecurityReviewer and Reviewer in PARALLEL when possible
- If any phase fails, explain what went wrong and propose a recovery path
- Never skip the security review
```

### Parallel Multi-Perspective Review

An orchestrator that runs multiple reviews simultaneously:

```markdown
---
name: PRReviewer
description: Reviews a PR from multiple perspectives in parallel.
tools: ['agent', 'search', 'read']
agents: ['SecurityReviewer', 'PerformanceReviewer', 'A11yReviewer']
model: 'Claude Opus 4.5'
---

# PR Review Orchestrator

Review the current changes from three perspectives simultaneously:

1. **Security** — delegate to SecurityReviewer
2. **Performance** — delegate to PerformanceReviewer
3. **Accessibility** — delegate to A11yReviewer

Run all three in parallel. Once all complete, consolidate findings into a
single review summary grouped by severity.
```

### Lightweight Orchestration (No Extra Agent Files)

You don't always need separate agent files. Shape subagent behavior through your prompt alone:

```
Review this codebase using three subagents, each with a different focus:

Subagent 1 — Security: Check for injection vulnerabilities, hardcoded
secrets, and missing auth checks.

Subagent 2 — Performance: Identify N+1 queries, unnecessary re-renders,
and missing caching opportunities.

Subagent 3 — Code Quality: Find duplicated logic, overly complex functions,
and missing error handling.

Run all three in parallel. Consolidate into a prioritized action plan.
```

This works because each subagent approaches the code fresh, without being anchored by what the other perspectives found.

---

## 7. Cross-Environment Delegation

Beyond subagents (which run in the same VS Code session), you can delegate across entirely different runtimes.

### Local → Copilot CLI (Background)

Hand off to a background agent for autonomous, long-running work:

1. Start with a local agent (interactive planning).
2. Select **Continue in Copilot CLI** from the session type dropdown.
3. VS Code creates a new session, carrying over conversation history.
4. Copilot CLI runs in a Git worktree (isolated from your workspace).
5. Review changes when done → **Apply** to merge into your workspace.

Use case: *Plan interactively, then let the CLI implement autonomously while you keep coding.*

### Local/CLI → Cloud Agent

Delegate to the GitHub-hosted Copilot coding agent for fully autonomous work:

**From Copilot CLI:**
```
/delegate Implement the authentication module per the plan above.
```

**From VS Code chat:**
- Use `#copilotCodingAgent` in your prompt
- Or select **Delegate to coding agent** from the action menu

**From TODO comments:**
```typescript
// TODO: Add input validation for all API endpoints
```
A Code Action appears → **Delegate to coding agent** → Copilot creates a PR.

**From GitHub Issues:**
- Assign the issue to `@copilot`
- Or mention `@copilot` in an issue comment

The cloud agent works in an isolated GitHub Actions environment, creates a PR, and you review it like any other PR. Guide it with `@copilot` comments.

### Decision Guide: Which Runtime?

| Scenario | Runtime | Why |
| :--- | :--- | :--- |
| Quick interactive changes | **Local** | Immediate feedback, full workspace access |
| Long-running implementation | **Copilot CLI** | Background, worktree isolation |
| PR-based collaboration | **Cloud** | GitHub integration, team review |
| Multiple independent tasks | **Parallel CLI sessions** | Each gets its own worktree |
| Research / exploration | **Subagent** | Context-isolated, disposable |

---

## 8. Design Patterns & Best Practices

### Separation of Concerns

The customization stack has four distinct layers that compose independently:

| Layer | Files | Changes |
| :--- | :--- | :--- |
| **Who** (agent persona) | `.agent.md` | Agent behavior, tools, model |
| **Rules** (project standards) | `copilot-instructions.md`, `*.instructions.md` | Coding conventions |
| **How** (capabilities) | `SKILL.md`, scripts, templates | Reusable procedures |
| **When** (deterministic actions) | `hooks/*.json` | Lifecycle automation |

You can update standards without rewriting agents. Upgrade skills without modifying instructions. Change agent behavior while keeping conventions intact.

### Tool Restriction Principles

```
Principle of Least Privilege:
  Planner     → read-only tools only (search, read, web)
  Implementer → edit tools, no web access
  Reviewer    → read-only + terminal (for running tests/lint)
  Orchestrator → 'agent' tool + read-only (delegates everything)
```

### Model Selection Strategy

Match the model to the task complexity and cost:

```yaml
# Orchestrator: needs strong reasoning for coordination
model: 'Claude Opus 4.5'

# Planner: needs deep understanding for architecture
model: ['Claude Opus 4.5', 'GPT-5.2']

# Implementer: strong coding, fast
model: 'Claude Sonnet 4.5'

# Research subagent: fast, cheap, high volume
model: 'Claude Haiku 4.5'

# Security reviewer: needs strong reasoning
model: 'Claude Opus 4.5'
```

### Context Management Tips

- **Use subagents for research** — keep the main agent's context window clean.
- **One subagent = one job** — don't overload a subagent with multiple unrelated tasks.
- **Use `/compact` when context grows** — e.g., `/compact keep only the implementation plan`.
- **Subagent prompts should be self-contained** — include all necessary context since they don't inherit conversation history.
- **Prefer parallel subagents** when tasks are independent — saves time and tokens.

### Naming Conventions

```text
.github/agents/
├── orchestrators/
│   ├── feature-builder.agent.md      # End-to-end feature orchestrator
│   └── pr-reviewer.agent.md          # Multi-perspective PR review
├── workers/
│   ├── planner.agent.md              # Read-only planning
│   ├── implementer.agent.md          # Code editing
│   ├── reviewer.agent.md             # Code review
│   └── security-reviewer.agent.md    # Security scanning
└── internal/
    ├── researcher.agent.md           # Subagent-only (user-invocable: false)
    └── architect.agent.md            # Subagent-only
```

> Note: The subfolder structure is for human organization. VS Code discovers all `*.agent.md` files recursively in `.github/agents/`.

---

## 9. Complete Multi-Agent Setup (Starter Kit)

Here's a ready-to-use set of agents that work together. Copy into `.github/agents/`.

### `planner.agent.md`
```markdown
---
name: Planner
description: Produces implementation plans. Read-only — never edits files.
tools: ['search/codebase', 'search/usages', 'read_file', 'web/fetch']
model: ['Claude Opus 4.5', 'GPT-5.2']
handoffs:
  - label: Implement Plan
    agent: Implementer
    prompt: "Implement the plan above. Follow all repo conventions."
    send: false
  - label: Write Tests First
    agent: TDDWriter
    prompt: "Write failing tests for the plan above."
    send: false
---

You are a planning-only agent. Analyze the request and produce a detailed
implementation plan with file paths, steps, risks, and testing strategy.
Never edit files.
```

### `implementer.agent.md`
```markdown
---
name: Implementer
description: Implements code changes per a plan.
tools: ['edit_file', 'create_file', 'delete_file', 'terminal',
        'search/codebase', 'read_file']
model: 'Claude Sonnet 4.5'
handoffs:
  - label: Review Changes
    agent: Reviewer
    prompt: "Review all changes made in this session."
    send: false
---

You implement plans step-by-step. After each change, verify it compiles.
If a step is unclear, STOP and ask — do not guess.
```

### `reviewer.agent.md`
```markdown
---
name: Reviewer
description: Reviews code changes for quality, correctness, and test coverage.
tools: ['search/codebase', 'read_file', 'terminal']
model: 'Claude Opus 4.5'
---

You are a code reviewer. Check for:
1. Correctness — does the code do what the plan specified?
2. Quality — naming, structure, duplication, error handling
3. Tests — are there tests? Do they pass? (`pnpm test`)
4. Conventions — does it follow project standards?

Output a structured review with severity levels.
```

### `security-reviewer.agent.md`
```markdown
---
name: SecurityReviewer
description: Scans code for security vulnerabilities.
user-invocable: false
tools: ['search/codebase', 'read_file', 'terminal']
model: 'Claude Opus 4.5'
---

You are a security specialist. Scan for OWASP Top 10 vulnerabilities.
Run `npm audit` when relevant. Output findings with severity and fix recommendations.
```

### `orchestrator.agent.md`
```markdown
---
name: Orchestrator
description: >-
  Coordinates feature development across planning, implementation, security
  review, and code review using specialized subagents.
tools: ['agent', 'search', 'read']
agents: ['Planner', 'Implementer', 'SecurityReviewer', 'Reviewer']
model: 'Claude Opus 4.5'
---

# Feature Development Orchestrator

You coordinate end-to-end feature development. You NEVER write code yourself.

## Workflow
1. Delegate to **Planner** → get implementation plan
2. Present plan to user → wait for approval
3. Delegate to **Implementer** → execute the plan
4. Delegate to **SecurityReviewer** and **Reviewer** in parallel
5. If issues found → delegate back to **Implementer** with fix list
6. Present final summary

## Rules
- Always get user approval before Phase 3 (implementation)
- Run security + code review in parallel
- Never skip the security review
- If any phase fails, explain what went wrong and propose recovery
```

---

## 10. Comparison: Handoffs vs Subagents vs Cloud Delegation

| Dimension | Handoffs | Subagents | Cloud Delegation |
| :--- | :--- | :--- | :--- |
| **Who decides** | User clicks button | Agent decides autonomously | User triggers explicitly |
| **Context** | Conversation transfers (shared history) | Isolated (fresh context) | Full issue/PR context |
| **Parallelism** | Sequential only | Multiple in parallel | One per issue/PR |
| **Review gate** | Built-in (`send: false`) | No gate — auto-returns result | PR review process |
| **Best for** | Plan → implement → review chains | Research, analysis, focused subtasks | Autonomous background work |
| **Setup** | `handoffs:` in frontmatter | `tools: ['agent']`, `agents: [...]` | Assign issue to `@copilot` |
| **Nesting** | Chain of any length | No nesting (1 level deep) | N/A |
| **Undo** | Switch back to previous agent | N/A — result is already returned | Close PR |

---

## 11. Troubleshooting

| Problem | Cause | Fix |
| :--- | :--- | :--- |
| Agent not in picker | `user-invocable: false` or file not in `.github/agents/` | Check frontmatter and file location |
| Subagent not invoked | `agent` tool not in `tools` list | Add `'agent'` to tools array |
| Subagent uses wrong agent | `agents` property doesn't include the name | Add agent name to `agents` array |
| Handoff button doesn't appear | Frontmatter syntax error or `handoffs` misspelled | Validate YAML syntax |
| Agent can edit files when it shouldn't | Tools not restricted | Explicitly whitelist read-only tools |
| Model not available | Model name incorrect or not in subscription | Check model name format: `'Model (vendor)'` |
| Hooks not firing for agent | Hooks defined only for different event | Check hook event names match (PascalCase in VS Code) |
| Agent not found in cloud | Missing `target` or file not on default branch | Ensure agent file is merged to main/default |

### Debugging Agent Behavior

Use the **Agent Debug Panel** (`Chat: Open Agent Debug View`) to see:
- Which customizations are loaded (agents, instructions, skills, hooks)
- System prompts and tool calls in real-time
- Subagent spawning and result return
- Hook execution and output

---

## 12. Key Resources

- **VS Code Subagents Docs:** [code.visualstudio.com/docs/copilot/agents/subagents](https://code.visualstudio.com/docs/copilot/agents/subagents)
- **Custom Agents Docs:** [code.visualstudio.com/docs/copilot/customization/custom-agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents)
- **Cloud Agent Custom Agents:** [docs.github.com/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents)
- **Agent Orchestration Blog:** [blog.iamarinroy.com — Orchestrating GitHub Copilot Custom Agents](https://blog.iamarinroy.com/2026/02/orchestrating-github-copilot-custom.html)
- **Community Examples:** [github.com/github/awesome-copilot](https://github.com/github/awesome-copilot)
- **Multi-Agent Blog Post:** [code.visualstudio.com/blogs/2026/02/05/multi-agent-development](https://code.visualstudio.com/blogs/2026/02/05/multi-agent-development)
