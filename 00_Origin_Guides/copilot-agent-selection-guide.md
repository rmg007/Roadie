# VS Code Copilot — Local, Background & Cloud Agent Selection Guide
**Updated:** April 2026 · **Covers:** VS Code 1.110+, Copilot CLI, Copilot Cloud Agent  
**Purpose:** Decision framework an AI coding agent (or developer) uses to pick the right execution environment for each task.

---

## 1. The Three Execution Environments

VS Code agents run in three environments. The key dimensions are **where** the agent runs and **how you interact** with it.

| | **Local** | **Background (Copilot CLI)** | **Cloud (Coding Agent)** |
| :--- | :--- | :--- | :--- |
| **Runs on** | Your machine, inside VS Code | Your machine, outside VS Code process | GitHub's infrastructure (Actions runners) |
| **Interaction** | Interactive — you steer in real time | Autonomous — runs while you keep coding | Fully autonomous — runs while you're away |
| **Workspace access** | Full — all VS Code tools, MCP servers, extensions, selections, terminal | Partial — local files + MCP, but no VS Code built-in tools | None — reads repo from GitHub, no local tools or context |
| **Isolation** | None — edits your working directory directly | Optional Git worktree — changes isolated from your workspace | Always isolated — creates a `copilot/` branch |
| **Persistence** | Stops if VS Code closes | Survives VS Code restarts | Runs independently of your machine |
| **Parallelism** | One active session at a time | Multiple parallel sessions (each in its own worktree) | Multiple parallel sessions (each on its own runner) |
| **Output** | Direct file edits you review inline | Diff view → "Apply" to merge into workspace | Draft Pull Request on GitHub |
| **Collaboration** | Solo — only you see the changes | Solo — but can `/delegate` to cloud | Team — PR workflow with reviews, CI, comments |
| **Best models** | All — full model picker access | CLI-available models | Models configured in cloud agent service |
| **Custom agents** | ✅ Full support (tools, handoffs, subagents) | ✅ Supports instructions, skills, hooks | ✅ Supports custom agents, MCP, hooks |
| **Cost** | Premium requests per your prompt (tool calls are free) | Same billing as local | Same billing — each prompt counts |

---

## 2. The Decision Framework

An agent (or developer) should pick the environment based on **four factors**: interactivity need, scope of change, isolation requirement, and collaboration need.

### Primary Decision Tree

```
DOES THE TASK NEED REAL-TIME STEERING?
│
├── YES (exploratory, iterative, unclear scope, needs feedback)
│   └── LOCAL
│       Examples:
│       - Debugging with breakpoints and live terminal output
│       - Exploring a new codebase ("what does this do?")
│       - Iterating on UI with live preview
│       - Tasks that need VS Code tools (test runner, debugger, browser)
│       - Prototyping where you'll change direction frequently
│
└── NO (scope is clear, context is sufficient)
    │
    ├── DOES IT NEED A PR / TEAM REVIEW?
    │   │
    │   ├── YES
    │   │   └── CLOUD
    │   │       Examples:
    │   │       - Feature implementation that teammates will review
    │   │       - Bug fix assigned via GitHub issue
    │   │       - Changes that should go through CI before merging
    │   │       - Cross-service refactors that need architectural review
    │   │       - Tasks you want to "fire and forget"
    │   │
    │   └── NO (changes are for you, not a PR)
    │       │
    │       ├── DO YOU WANT TO KEEP CODING WHILE IT RUNS?
    │       │   │
    │       │   ├── YES
    │       │   │   └── BACKGROUND (Copilot CLI)
    │       │   │       Examples:
    │       │   │       - Implementing a plan you already approved
    │       │   │       - Multi-file rename/refactor with clear scope
    │       │   │       - Writing test suites for existing code
    │       │   │       - Generating boilerplate across many files
    │       │   │       - Running multiple independent tasks in parallel
    │       │   │
    │       │   └── NO (happy to wait, want maximum tool access)
    │       │       └── LOCAL
    │       │
    │       └── IS THE TASK RISKY? (might break things, want isolation)
    │           │
    │           ├── YES → BACKGROUND with worktree isolation
    │           └── NO → LOCAL (faster, simpler)
```

### Quick Reference Rules

| Signal in the task | Route to | Why |
| :--- | :--- | :--- |
| "fix this bug" + you're looking at the file | **Local** | Needs your context (selection, terminal, debugger) |
| "explain this code" / "what does X do?" | **Local** | Interactive Q&A, no file changes |
| "implement this plan" (plan already exists) | **Background** | Well-scoped, you want to keep working |
| "write tests for all files in /src" | **Background** | Bulk work, parallelizable, clear scope |
| "refactor auth across all services" | **Cloud** | Large scope, needs team review via PR |
| GitHub issue assigned to `@copilot` | **Cloud** | Already in GitHub's workflow |
| "I want to try two approaches and compare" | **Background ×2** | Parallel worktrees, compare diffs |
| TODO comment in code | **Cloud** | Code Action → delegate to coding agent → PR |
| "redesign the API" (unclear scope) | **Local** (Plan agent first) | Needs interactive planning before execution |

---

## 3. Local Agents — Interactive Development

### When to Use

- You need **immediate feedback** — see changes as they happen
- The task requires **VS Code tools** — debugger, test runner, integrated browser, terminal output, text selections, problem markers
- You're **exploring or iterating** — changing direction based on results
- You need **extension-provided tools** — linters, formatters, language servers
- The scope is **unclear** — you'll refine as you go

### Built-in Agent Personas

| Persona | What It Does | When to Use |
| :--- | :--- | :--- |
| **Agent** | Full autonomous agent — plans, edits files, runs commands, self-corrects | Default for most implementation tasks |
| **Plan** | Creates structured implementation plans without editing files | Before handing off to Background or Cloud |
| **Ask** | Answers questions without making changes | Code explanations, architecture questions |

### Third-Party Local Agents

| Agent | Strengths | When to Prefer |
| :--- | :--- | :--- |
| **Claude** (Anthropic) | Strong reasoning, clean code, design-sensitive | Frontend work, complex debugging, architecture |
| **Codex** (OpenAI) | Code-optimized, fast iteration | Rapid implementation, bulk code generation |

Select from the agent picker dropdown in the Chat view.

### Handoff Patterns from Local

| From Local → | How | When |
| :--- | :--- | :--- |
| **Background** | Session type dropdown → "Copilot CLI" | Plan is ready, you want to keep coding |
| **Cloud** | Session type dropdown → "Cloud" | Task needs a PR / team review |
| **Cloud (from Plan)** | "Continue in Cloud" button | Plan agent → cloud implementation |

### Tips

- Start with **Plan** persona when scope is unclear — get a plan, review it, *then* hand off to Background or Cloud for execution.
- Use **subagents** for research tasks within a local session to keep your main context clean.
- Local agents have full access to **all VS Code built-in tools** — this is their unique advantage over Background and Cloud.

---

## 4. Background Agents (Copilot CLI) — Autonomous Local Execution

### When to Use

- Task has a **well-defined scope** and all necessary context
- You want to **keep coding** in your main workspace while the agent works
- You want **isolation** via Git worktrees (changes don't touch your working directory)
- You need to run **multiple independent tasks in parallel**
- You want to **experiment safely** — worktree means easy discard if results are bad

### How It Works

1. Start a Copilot CLI session from the Chat view (session type → "Copilot CLI")
2. Choose isolation mode:
   - **Worktree** (recommended) — agent works in a separate Git worktree. Your workspace stays untouched. Changes are applied via "Apply" after review.
   - **Workspace** — agent works directly in your workspace (same as local, but runs in background process)
3. Agent runs autonomously. You can send follow-up prompts to steer it.
4. Review changes via diff view when done → Apply or Discard.

### Worktree vs Workspace Isolation

| | Worktree | Workspace |
| :--- | :--- | :--- |
| Your files during execution | Untouched | Being modified |
| Risk of conflicts | None | Possible if you edit same files |
| Permission level | Auto-approved (Bypass Approvals) | Configurable (Default/Bypass/Autopilot) |
| Best for | Safe parallel work, experiments | Quick tasks where isolation isn't needed |

### Parallel Sessions

You can run **multiple Copilot CLI sessions simultaneously**, each in its own worktree:

```
Session 1: "Add authentication to the API"     → worktree-1/
Session 2: "Write integration tests for /users" → worktree-2/
Session 3: "Refactor database connection pool"  → worktree-3/
```

All three run concurrently. Review and apply each independently.

### Delegation from Background

```
/delegate Implement the feature per the plan above
```

This hands off the current Background session to the Cloud Agent, carrying over full conversation context. The cloud agent creates a PR. Use when you realize the task needs team review or CI.

### Limitations

- **No VS Code built-in tools** — can't access debugger, test runner UI, browser tools, or extension-provided tools
- **Limited MCP server access** — only local MCP servers that don't require authentication
- **Model access** — limited to models available via the CLI (may not include all IDE models)

### Tips

- **Commit before starting** — the worktree is based on your current Git state
- Use **`/compact`** for long sessions to manage context
- Use **`/autoApprove`** or **`/yolo`** to toggle auto-approval if you trust the agent
- Background sessions **survive VS Code restarts** — check the Sessions view if you closed VS Code during a task

---

## 5. Cloud Agents (Copilot Coding Agent) — Fully Autonomous + PR Workflow

### When to Use

- Task should produce a **Pull Request** for team review
- You want to **walk away** — the agent runs on GitHub's infrastructure
- The task benefits from **CI/CD integration** — automated tests, CodeQL, dependency scanning
- You're working from a **GitHub Issue** — assign to `@copilot`
- Changes need **collaboration** — teammates can comment on the PR, the agent responds

### How It Works

1. **Trigger** (any of these):
   - VS Code: Session type → "Cloud" in Chat view
   - VS Code: `#copilotCodingAgent` tool in your prompt
   - VS Code: "Delegate to coding agent" action / Code Action on TODO comments
   - GitHub.com: Assign issue to `@copilot`
   - GitHub.com: Comment `@copilot` on a PR
   - Copilot CLI: `/delegate` command
2. **Agent runs on GitHub Actions runner** — reads your repo, creates a `copilot/` branch
3. **Agent implements changes** autonomously — edits files, runs tests, self-corrects
4. **Security checks** run — CodeQL, dependency scanning, secret scanning
5. **Draft PR created** — you and your team review
6. **Iterate via PR comments** — tag `@copilot` to request changes; agent responds and updates

### Entry Points Summary

| Entry Point | How | When |
| :--- | :--- | :--- |
| VS Code Chat | Session type dropdown → Cloud | Interactive start with context |
| Plan agent | "Continue in Cloud" button | After planning locally |
| Copilot CLI | `/delegate` command | Escalate from background |
| GitHub Issue | Assign to `@copilot` | Task already tracked as an issue |
| PR comment | `@copilot please fix the failing test` | Agent addresses review feedback |
| TODO comment | Code Action → "Delegate to coding agent" | Quick delegation from code |

### What Cloud Agents CAN'T Do

- **No VS Code tools** — no debugger, no terminal output, no integrated browser, no test runner UI
- **No local file access** — only sees what's committed to the repo
- **No extension tools** — can't use VS Code extensions
- **No real-time steering** — you can comment on the PR, but it's async
- **Limited MCP** — only MCP servers configured in repository settings (not local `.vscode/mcp.json`)

### What Cloud Agents ARE Best At

- Implementing well-scoped features from issue descriptions
- Addressing code review feedback on existing PRs
- Running security scans (CodeQL) as part of the workflow
- Working while you sleep / work on other things
- Producing auditable output (PR history, CI results)

### Cloud Agent Configuration

Customize via `.github/copilot-instructions.md`, `.github/agents/*.agent.md`, `.github/skills/`, `.github/hooks/`, and the `copilot-setup-steps.yml` workflow. See the Customization Reference.

---

## 6. The Recommended Workflow: Plan → Background → Cloud

The most effective pattern combines all three:

```
┌─────────────────────────────────────────────────────────┐
│ PHASE 1: LOCAL (Plan agent)                             │
│ - Clarify requirements interactively                    │
│ - Create a structured implementation plan               │
│ - Review and approve the plan                           │
│                    │                                    │
│                    ▼                                    │
│ PHASE 2: BACKGROUND (Copilot CLI, worktree isolation)   │
│ - Implement the approved plan autonomously              │
│ - You continue coding in your main workspace            │
│ - Review diffs when done                                │
│                    │                                    │
│            ┌───────┴───────┐                            │
│            │               │                            │
│            ▼               ▼                            │
│   Changes look good    Needs team review                │
│     → Apply locally      → /delegate to Cloud           │
│                              → PR created               │
│                              → Team reviews             │
│                              → CI runs                  │
│                              → Merge                    │
└─────────────────────────────────────────────────────────┘
```

**Why this works:**
- **Plan locally** — you catch misunderstandings early, when they're cheap to fix
- **Implement in background** — you stay productive while the agent works
- **Escalate to cloud only when needed** — saves cloud compute for tasks that genuinely need PR workflow

---

## 7. Building a Router Agent (No Extension Needed)

Your draft proposed a custom VS Code extension. You don't need one — a custom `.agent.md` file with the right instructions achieves the same routing behavior using built-in mechanisms.

### `.github/agents/router.agent.md`

```markdown
---
name: Router
description: >-
  Analyzes tasks and recommends the best execution environment
  (Local, Background, or Cloud). Does not implement — only routes.
tools: ['search/codebase', 'read_file', 'search/usages']
handoffs:
  - label: Run Locally (Agent mode)
    agent: agent
    prompt: "Implement the task described above."
    send: false
  - label: Run in Background (Copilot CLI)
    agent: agent
    prompt: "Implement the task described above."
    send: false
  - label: Delegate to Cloud (PR workflow)
    agent: agent
    prompt: "Implement the task described above."
    send: false
---

# Task Router

You analyze a task request and recommend the best execution environment.

## Investigation Steps

1. **Identify the scope** — use `search/codebase` and `search/usages` to estimate
   how many files will be affected.
2. **Check complexity** — is this a simple change or does it span multiple modules/services?
3. **Assess interactivity need** — does the user need to steer this, or is the scope clear?
4. **Check collaboration need** — should this produce a PR for team review?

## Routing Rules

Route to **LOCAL** when:
- Task needs real-time feedback (debugging, exploring, UI iteration)
- Scope is unclear — needs interactive planning first
- Task requires VS Code tools (debugger, browser, test runner)
- User is actively watching and wants to steer
- Fewer than ~5 files affected and straightforward

Route to **BACKGROUND** when:
- Scope is well-defined (a plan exists or the task is clear)
- Multi-file changes but no PR needed
- User wants to keep working on something else
- Task is parallelizable (multiple independent changes)
- Experimentation — try an approach in an isolated worktree

Route to **CLOUD** when:
- Changes should go through a Pull Request
- Task was assigned as a GitHub Issue
- Cross-service or architectural changes needing team review
- User wants to "fire and forget"
- CI/CD pipeline should validate the changes

## Output

State your recommendation clearly:
1. **Environment:** LOCAL / BACKGROUND / CLOUD
2. **Reasoning:** 2-3 sentences explaining why
3. **Suggest the handoff** — the user can click the appropriate button below

If the scope is unclear, recommend LOCAL with the Plan agent first.
```

### How to Use

1. Select **Router** from the agents dropdown
2. Describe your task: *"Optimize the database queries across all services"*
3. Router investigates the codebase, estimates scope, and recommends an environment
4. Click the appropriate handoff button to proceed
5. Switch to the recommended session type manually (session type dropdown)

This gives you the same intelligent routing behavior as the extension approach, but using built-in mechanisms that already exist, with zero custom code to maintain.

---

## 8. Parallel Execution Strategies

### Strategy 1: Multiple Background Sessions

Run several independent tasks simultaneously:

```
Session A (worktree): "Add rate limiting to the API"
Session B (worktree): "Write E2E tests for the checkout flow"
Session C (worktree): "Update all deprecated dependency imports"
```

Each runs in its own worktree. Review and apply independently. No conflicts.

### Strategy 2: Background + Cloud in Parallel

```
Background session: Implement a feature locally for your own testing
Cloud session:      Same feature as a PR for the team to review
```

Compare approaches. Pick the better one.

### Strategy 3: Local + Multiple Backgrounds

```
Local session:       You're debugging a critical bug interactively
Background 1:        Implementing a feature from the sprint backlog
Background 2:        Writing missing tests for the auth module
```

You stay productive on the critical bug while routine work runs autonomously.

### Strategy 4: Subagents for Parallel Research (within any session)

```
Main agent prompt:
"Analyze this codebase for refactoring opportunities. Use subagents to:
1. Find duplicate code patterns
2. Identify unused exports
3. Check for security vulnerabilities
Run all three in parallel."
```

Subagents run concurrently within a single session. Results consolidate into the main agent's context.

---

## 9. Common Mistakes & How to Avoid Them

| Mistake | Why It's Bad | Fix |
| :--- | :--- | :--- |
| Using Cloud for every task | Slow feedback loop, unnecessary PR overhead | Start local. Only go to Cloud when a PR is needed. |
| Using Local for bulk work | Blocks your editor, can't work on other things | Use Background for well-scoped bulk tasks. |
| Background without worktree isolation | Agent edits your files while you're working — conflicts | Always use worktree mode unless you're sure there's no overlap. |
| Skipping the Plan phase | Agent misunderstands requirements, wastes time | Use Plan agent locally first, then hand off. |
| Not committing before Background | Worktree is based on your Git state — uncommitted changes are lost | Always commit (or stash) before starting a Background session. |
| Cloud without `copilot-instructions.md` | Cloud agent has no context about your project conventions | Always have a `.github/copilot-instructions.md` in your repo. |
| Sending many small prompts on premium models | Each prompt costs PRs at the model's multiplier | Batch instructions into fewer, more detailed prompts. |

---

## 10. Environment Selection for Agent Definitions

When building multi-agent systems with `.agent.md` files, you can target specific environments:

```yaml
# Agent that only runs in VS Code (local/background)
---
name: LocalDebugger
target: vscode
tools: ['terminal', 'search/codebase', 'read_file', 'edit_file']
---

# Agent that only runs in the cloud
---
name: PRImplementer
target: github-copilot
tools: ['read', 'edit', 'search']
---
```

The `target` property ensures agents only appear in environments where their tools are available. A debugger agent makes no sense in the cloud (no debugger access). A PR-focused agent makes no sense locally.

---

## 11. Quick Decision Card

```
╔══════════════════════════════════════════════════════════════╗
║  ENVIRONMENT ROUTING — paste into agent instructions         ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  LOCAL when:                                                 ║
║    • Scope is unclear → start with Plan agent                ║
║    • Need VS Code tools (debugger, browser, test runner)     ║
║    • Interactive exploration or iteration                    ║
║    • Quick single-file fixes                                 ║
║                                                              ║
║  BACKGROUND (Copilot CLI) when:                              ║
║    • Scope is clear + you want to keep coding                ║
║    • Multi-file changes that don't need a PR                 ║
║    • Parallel independent tasks (multiple worktrees)         ║
║    • Experimental/risky changes (worktree = safe discard)    ║
║                                                              ║
║  CLOUD when:                                                 ║
║    • Changes need a Pull Request for team review             ║
║    • Task is a GitHub Issue → assign to @copilot             ║
║    • Fire-and-forget (runs while you're away)                ║
║    • CI/CD validation needed before merging                  ║
║                                                              ║
║  HANDOFFS:                                                   ║
║    Local → Background: session type dropdown → Copilot CLI   ║
║    Local → Cloud: session type dropdown → Cloud              ║
║    Plan → Cloud: "Continue in Cloud" button                  ║
║    Background → Cloud: /delegate command                     ║
║    TODO → Cloud: Code Action → Delegate to coding agent      ║
║    Issue → Cloud: assign to @copilot on GitHub               ║
║                                                              ║
║  DEFAULT: Start LOCAL. Escalate as needed.                   ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 12. Key Resources

- **Agents Overview:** [code.visualstudio.com/docs/copilot/agents/overview](https://code.visualstudio.com/docs/copilot/agents/overview)
- **Background Agents:** [code.visualstudio.com/docs/copilot/agents/background-agents](https://code.visualstudio.com/docs/copilot/agents/background-agents)
- **Cloud Agents:** [code.visualstudio.com/docs/copilot/agents/cloud-agents](https://code.visualstudio.com/docs/copilot/agents/cloud-agents)
- **Agents Tutorial:** [code.visualstudio.com/docs/copilot/agents/agents-tutorial](https://code.visualstudio.com/docs/copilot/agents/agents-tutorial)
- **Multi-Agent Blog:** [code.visualstudio.com/blogs/2026/02/05/multi-agent-development](https://code.visualstudio.com/blogs/2026/02/05/multi-agent-development)
- **Copilot Coding Agent:** [code.visualstudio.com/docs/copilot/copilot-coding-agent](https://code.visualstudio.com/docs/copilot/copilot-coding-agent)
