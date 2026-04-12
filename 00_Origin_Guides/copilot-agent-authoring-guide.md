# Writing Agents for VS Code Copilot — Complete Authoring Guide
**Updated:** April 2026 · **Covers:** VS Code 1.110+, Copilot CLI, Copilot Cloud Agent  
**Companion guides:** Customization Reference, Orchestration & Delegation Guide, Skill Authoring Guide

---

## 1. Two Ways to Create Agents

VS Code supports two fundamentally different approaches to creating agents. Choose based on what you need:

| | **Custom Agent Files (`.agent.md`)** | **Extension Chat Participants (TypeScript)** |
| :--- | :--- | :--- |
| **Complexity** | Zero code — just Markdown + YAML | Full VS Code extension (TypeScript, npm, packaging) |
| **Creation time** | Minutes | Hours to days |
| **What it does** | Replaces the model's identity, restricts tools, defines handoffs | Full control: custom prompt construction, tool orchestration, arbitrary VS Code API access |
| **Portability** | Works across VS Code, CLI, Cloud Agent, JetBrains | VS Code only |
| **Distribution** | Commit to repo (`.github/agents/`) or user profile | VS Code Marketplace |
| **Best for** | Personas, workflows, orchestration, tool restriction | Deep IDE integration, custom UX, complex tool chains, proprietary logic |
| **Examples** | Planner, Reviewer, Security Auditor, Orchestrator | @workspace, @terminal, @vscode (built-in), custom DB explorers |

**Rule of thumb:** Start with `.agent.md`. Only move to a Chat Participant extension if you need something `.agent.md` can't do (custom UI, direct VS Code API access, proprietary tool orchestration, marketplace distribution).

---

## 2. Custom Agent Files (`.agent.md`) — The No-Code Approach

This is what most developers need. A custom agent is a Markdown file with YAML frontmatter that changes **who the model is** — its persona, available tools, model preference, and workflow transitions.

### 2.1 Complete Frontmatter Reference

```yaml
---
# === Identity ===
name: string                    # Unique ID. Defaults to filename without .agent.md
description: string             # Required. What the agent does. Shown in picker + placeholder text.
argument-hint: string           # Optional. Hint shown in chat input when agent is selected.

# === Capabilities ===
tools: string[]                 # Tool whitelist. Omit = all tools. [] = no tools.
                                #   Built-in: edit_file, create_file, delete_file, terminal,
                                #             search/codebase, search/usages, read_file, web/fetch
                                #   Tool sets: 'edit', 'read', 'search', 'agent'
                                #   MCP: '<server-name>/*' or '<server-name>/tool-name'
                                #   Include 'agent' to enable subagent delegation.

agents: string[]                # Subagent whitelist. '*' = all. [] = none.
                                # Names must match other .agent.md `name` fields.
                                # Requires 'agent' in tools list.

model: string | string[]        # Model or fallback order.
                                #   Format: 'Model Name' or 'Model Name (vendor)'
                                #   Examples: 'Claude Opus 4.5', 'GPT-5.2', 'GPT-5 mini'
                                #   Array: tries each in order until one is available.

# === MCP Servers (scoped to this agent) ===
mcp-servers:
  server-name:
    type: local | stdio | http | sse
    command: string
    args: string[]
    tools: string[]             # Tool allowlist. ['*'] = all.
    env:
      KEY: ${{ secrets.COPILOT_MCP_KEY }}

# === Workflow ===
handoffs:                       # Sequential transitions between agents
  - label: string               # Button text
    agent: string               # Target agent name
    prompt: string              # Pre-filled prompt
    send: bool                  # true = auto-send, false = user reviews
    model: string               # Optional model override

hooks:                          # Inline lifecycle hooks
  PostToolUse:
    - type: command
      command: string

# === Visibility ===
target: vscode | github-copilot # Restrict to one environment. Omit = both.
user-invocable: bool            # Default: true. false = hidden from picker (subagent-only).
disable-model-invocation: bool  # Default: false. true = can't be auto-invoked as subagent.
---

# Markdown body (max 30,000 characters)
# This is the agent's prompt — it defines behavior, rules, and workflow.
```

### 2.2 File Locations

| Location | Scope | How to Create |
| :--- | :--- | :--- |
| `.github/agents/{name}.agent.md` | Workspace (shared via git) | Manual or `Chat: New Custom Agent` command |
| User profile folder | User (all workspaces) | Chat Customizations editor → "New Agent (User)" |

> Configure extra locations via `chat.agentFilesLocations`. In monorepos, enable `chat.useCustomizationsInParentRepositories`.

### 2.3 Design Principles

**Principle of Least Privilege:** Only grant tools the agent actually needs.

```yaml
# ❌ Bad — agent can do everything, including destructive operations
tools: []  # (omitted = all tools)

# ✅ Good — planner can only read, not edit
tools: ['search/codebase', 'search/usages', 'read_file', 'web/fetch']
```

**Single Responsibility:** Each agent does one thing well.

```yaml
# ❌ Bad — one agent that plans, implements, reviews, and deploys
# ✅ Good — separate agents for each role, connected via handoffs/subagents
```

**Explicit Over Implicit:** State what the agent should NOT do.

```markdown
## Rules
- NEVER edit files — you are a planning-only agent
- NEVER run destructive terminal commands (rm, drop, delete)
- If requirements are ambiguous, ASK — do not guess
```

### 2.4 Complete Working Examples

#### Read-Only Planner

```markdown
---
name: Planner
description: Creates detailed implementation plans without editing files.
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

You are a planning-only agent. Analyze feature requests and produce
implementation plans. You NEVER edit files or run commands.

## Output Format
1. **Scope** — which files change and why
2. **Steps** — numbered, with specific file paths and function names
3. **Risks** — breaking changes, edge cases, perf concerns
4. **Testing** — what tests to add or update

If the request is ambiguous, ask clarifying questions BEFORE planning.
```

#### Orchestrator with Subagents

```markdown
---
name: FeatureBuilder
description: Coordinates feature development across planning, implementation, and review.
tools: ['agent', 'search', 'read']
agents: ['Planner', 'Implementer', 'SecurityReviewer', 'Reviewer']
model: 'Claude Opus 4.5'
---

You coordinate end-to-end feature development. You NEVER write code yourself.

## Workflow
1. Delegate to **Planner** → get implementation plan
2. Present plan to user → wait for approval
3. Delegate to **Implementer** → execute the approved plan
4. Delegate to **SecurityReviewer** and **Reviewer** in parallel
5. If issues found → delegate back to **Implementer** with fix list
6. Present final summary

## Rules
- Always get user approval before implementation
- Run security + code review in parallel when possible
- Never skip the security review
```

#### Agent with MCP Server

```markdown
---
name: DBExplorer
description: Explores database schema and queries using the project database.
tools: ['read_file', 'search/codebase', 'db-mcp/query', 'db-mcp/schema']
mcp-servers:
  db-mcp:
    type: stdio
    command: npx
    args: ['-y', '@example/db-mcp-server']
    tools: ['query', 'schema']
    env:
      DATABASE_URL: ${{ secrets.COPILOT_MCP_DATABASE_URL }}
---

You explore and explain database schemas. You can run read-only queries
to answer questions about data structure, relationships, and content.

## Rules
- ONLY run SELECT queries — never INSERT, UPDATE, DELETE, or DDL
- Always explain what a query does before running it
- Format results as tables when possible
```

#### Subagent-Only Agent (Hidden from Picker)

```markdown
---
name: InternalResearcher
description: Researches codebase patterns and returns analysis.
user-invocable: false
tools: ['search/codebase', 'read_file']
model: 'GPT-5 mini'
---

You research codebase patterns and return concise summaries.
Focus on answering the specific question — do not suggest changes.
```

### 2.5 Quick Creation Methods

| Method | Command |
| :--- | :--- |
| From Chat | Type in chat: *"create a custom agent for code review"* |
| From Command Palette | `Chat: New Custom Agent` |
| From Customizations Editor | Command Palette → `Chat: Open Chat Customizations` → Agents tab |
| From conversation | *"Create an agent from how we just debugged that auth issue"* |

---

## 3. Extension Chat Participants (TypeScript) — The Full-Control Approach

Use this when `.agent.md` isn't enough — you need custom prompt construction, direct VS Code API access, marketplace distribution, or complex tool orchestration.

### 3.1 When to Use Extensions Instead of `.agent.md`

| Need | `.agent.md` | Extension |
| :--- | :--- | :--- |
| Change model persona/tools | ✅ | ✅ |
| Handoffs between agents | ✅ | ✅ |
| Subagent delegation | ✅ | ✅ |
| Custom prompt construction with token budgeting | ❌ | ✅ |
| Access VS Code APIs (debugger, test runner, SCM) | ❌ | ✅ |
| Custom UI elements (webviews, tree views) | ❌ | ✅ |
| Proprietary tool orchestration logic | ❌ | ✅ |
| Marketplace distribution | ❌ | ✅ |
| Runs on Cloud Agent / CLI | ✅ | ❌ |

### 3.2 Architecture Overview

```
┌─────────────────────────────────────────────┐
│ VS Code Chat View                           │
│  User types: @myagent fix the login bug     │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Chat Participant (your extension)           │
│                                             │
│  1. Receive ChatRequest                     │
│  2. Build prompt (system + user + context)  │
│  3. Send to LLM via request.model           │
│  4. Process response (stream to chat)       │
│  5. Handle tool calls (if using tools)      │
│  6. Return result + metadata                │
└─────────────────────────────────────────────┘
```

### 3.3 Minimal Chat Participant

**`package.json` (manifest):**
```json
{
  "contributes": {
    "chatParticipants": [
      {
        "id": "myext.reviewer",
        "name": "reviewer",
        "description": "Reviews code for quality and security issues.",
        "isDefault": false
      }
    ]
  }
}
```

**`extension.ts` (handler):**
```typescript
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
  const participant = vscode.chat.createChatParticipant(
    'myext.reviewer',
    async (request, chatContext, stream, token) => {
      // 1. Build the prompt
      const messages = [
        vscode.LanguageModelChatMessage.System(
          'You are a code reviewer. Check for bugs, security issues, ' +
          'and style violations. Be specific about line numbers.'
        ),
        vscode.LanguageModelChatMessage.User(request.prompt)
      ];

      // 2. Include conversation history for multi-turn
      for (const turn of chatContext.history) {
        if (turn instanceof vscode.ChatResponseTurn) {
          const text = turn.response
            .filter(r => r instanceof vscode.ChatResponseMarkdownPart)
            .map(r => (r as vscode.ChatResponseMarkdownPart).value.value)
            .join('');
          messages.push(vscode.LanguageModelChatMessage.Assistant(text));
        }
      }

      // 3. Send request using the user's selected model
      //    (NEVER override with selectChatModels unless necessary)
      try {
        const response = await request.model.sendRequest(messages, {}, token);

        // 4. Stream the response
        for await (const fragment of response.text) {
          stream.markdown(fragment);
        }
      } catch (err) {
        if (err instanceof vscode.LanguageModelError) {
          if (err.code === 'quota') {
            stream.markdown('⚠️ Model quota exceeded. Try switching to GPT-5 mini.');
          } else {
            stream.markdown(`Error: ${err.message}`);
          }
        }
        throw err;
      }

      return { metadata: { command: '' } };
    }
  );

  participant.iconPath = vscode.Uri.joinPath(context.extensionUri, 'icon.png');
  context.subscriptions.push(participant);
}
```

### 3.4 Chat Participant with Tool Calling

For agents that need to call tools (MCP servers, built-in tools, custom tools):

```typescript
import * as vscode from 'vscode';
import * as chatUtils from '@vscode/chat-extension-utils';

export function activate(context: vscode.ExtensionContext) {
  const participant = vscode.chat.createChatParticipant(
    'myext.dbexplorer',
    async (request, chatContext, stream, token) => {
      // Use the chat-extension-utils library for tool orchestration
      const result = chatUtils.sendChatParticipantRequest(
        request,
        chatContext,
        {
          prompt: 'You are a database explorer. Help users understand ' +
                  'schema, relationships, and query patterns.',
          responseStreamOptions: {
            stream,
            references: true,
            responseText: true
          },
          // Filter tools to only those tagged for this participant
          tools: vscode.lm.tools.filter(
            tool => tool.tags.includes('myext-db')
          )
        },
        token
      );

      return await result.result;
    }
  );

  context.subscriptions.push(participant);
}
```

### 3.5 Registering Language Model Tools

Tools are atomic capabilities the LLM can invoke during conversation:

```typescript
// Register a tool that queries the database
vscode.lm.registerTool('myext-db-query', {
  tags: ['myext-db'],
  displayName: 'Query Database',
  description: 'Runs a read-only SQL query against the project database.',
  inputSchema: {
    type: 'object',
    properties: {
      sql: { type: 'string', description: 'The SQL SELECT query to run' }
    },
    required: ['sql']
  },
  async invoke(input: { sql: string }, token: vscode.CancellationToken) {
    // Validate: only allow SELECT
    if (!input.sql.trim().toUpperCase().startsWith('SELECT')) {
      return { content: [{ type: 'text', value: 'Only SELECT queries allowed.' }] };
    }

    const result = await runQuery(input.sql);
    return { content: [{ type: 'text', value: JSON.stringify(result, null, 2) }] };
  }
});
```

### 3.6 Advanced Prompt Construction with `@vscode/prompt-tsx`

For complex agents that need token budget management, use the TSX-based prompt library:

```typescript
import { renderPrompt, UserMessage, SystemMessage } from '@vscode/prompt-tsx';

// Priority-based pruning: lower priority content is dropped first
const { messages } = await renderPrompt(
  MyPromptComponent,
  { userQuery: request.prompt, files: relevantFiles },
  { modelMaxPromptTokens: request.model.maxInputTokens },
  request.model
);
```

Components use `flexGrow`, `flexReserve`, and `flexBasis` to control which content gets pruned first when the token budget is tight. Prioritize in this order:

1. System instructions (highest priority — never pruned)
2. Current user query
3. Tool results / workspace data
4. Last 1–2 turns of history
5. Older history (pruned first)

### 3.7 Response Stream API

The `ChatResponseStream` provides rich response capabilities:

| Method | Use Case |
| :--- | :--- |
| `stream.markdown(text)` | Primary response — text + code blocks |
| `stream.button({ title, command, args })` | Clickable action buttons |
| `stream.progress(message)` | Progress indicator for long operations |
| `stream.reference(uri)` | Link to a file or symbol |
| `stream.anchor(uri, title)` | Inline clickable link |
| `stream.filetree(tree, baseUri)` | Display directory structure |

```typescript
// Rich response example
stream.progress('Analyzing codebase...');
stream.markdown('## Review Results\n\n');
stream.markdown('Found 3 issues in the following files:\n');
stream.reference(vscode.Uri.file('/src/auth.ts'));
stream.markdown('\n**Fix suggestion:**\n```typescript\n// ...\n```\n');
stream.button({
  title: 'Apply Fix',
  command: 'myext.applyFix',
  arguments: [fixData]
});
```

### 3.8 Follow-Up Suggestions

Guide users to take the next logical step:

```typescript
participant.followupProvider = {
  provideFollowups(result, context, token) {
    return [
      { prompt: 'Fix all the issues found', label: 'Apply fixes' },
      { prompt: 'Write tests for the fixed code', label: 'Add tests' },
      { prompt: 'Explain the security implications', label: 'Explain risks' }
    ];
  }
};
```

---

## 4. Security Best Practices (Both Approaches)

### 4.1 Tool Restriction

```yaml
# .agent.md — restrict to read-only tools
tools: ['search/codebase', 'read_file']
```

```typescript
// Extension — filter tools by tag
tools: vscode.lm.tools.filter(t => t.tags.includes('safe-read-only'))
```

### 4.2 Secrets

- **Never** hardcode secrets in agent files or extension code
- `.agent.md`: Use `${{ secrets.COPILOT_MCP_* }}` for MCP server env vars
- Extensions: Use `vscode.SecretStorage` for credentials
- Block sensitive files from auto-approval: `chat.tools.edits.autoApprove` with `"**/.env": false`

### 4.3 Workspace Trust

- Agents are disabled in Restricted Mode
- Extensions should check `vscode.workspace.isTrusted` before performing operations
- Open untrusted repos in Restricted Mode first

### 4.4 Destructive Operations

```markdown
# In .agent.md body:
## Rules
- NEVER run commands that delete files, drop tables, or modify production data
- Always show the command you intend to run and wait for approval
- If `allowed-tools` includes terminal, only run commands explicitly listed in this prompt
```

```typescript
// In extension: require confirmation for destructive actions
const confirmed = await vscode.window.showWarningMessage(
  `This will delete ${files.length} files. Continue?`,
  'Yes', 'No'
);
if (confirmed !== 'Yes') return;
```

---

## 5. Debugging Agents

### Agent Debug Panel

Enable: `github.copilot.chat.agentDebugLog.enabled: true`

Shows in real-time:
- Which customizations are loaded (agents, instructions, skills, hooks)
- System prompts sent to the model
- Tool calls and their results
- Subagent spawning and results
- Hook execution

### Chat Debug View

Command Palette → `Chat: Open Chat Debug View`

Inspect:
- Exact system prompt and context
- Which files were included
- Token usage and pruning behavior
- Model selection

### Extension Debugging

For Chat Participant extensions:
- Use the **Run Extension** launch config in VS Code
- Set breakpoints in your handler
- Check the **Output** panel → "GitHub Copilot Chat" channel
- Use `console.log` in your handler (appears in Debug Console)

### Telemetry

Track response quality with `ChatResultFeedbackKind`:

```typescript
participant.onDidReceiveFeedback((feedback) => {
  if (feedback.kind === vscode.ChatResultFeedbackKind.Unhelpful) {
    // Log for analysis — improve prompts based on failure patterns
    telemetry.trackEvent('agent_feedback', {
      helpful: false,
      command: feedback.result.metadata?.command
    });
  }
});
```

---

## 6. Context Engineering

### Instruction Priority (highest → lowest)

1. **Personal** — VS Code user settings / user-level agent files
2. **Repository path-specific** — `.github/instructions/*.instructions.md` (via `applyTo`)
3. **Repository-wide** — `.github/copilot-instructions.md`
4. **Agent-specific** — `AGENTS.md` (root)
5. **Organization-level** — GitHub org settings

Higher-priority instructions win when conflicts exist.

### Context Management Tips

- **Keep `.agent.md` bodies focused** — only include what the model can't infer from the codebase
- **Use progressive disclosure** — start with minimal instructions, add rules only when you observe failures
- **Avoid context pollution** — if an agent does multiple things, split into separate agents with handoffs
- **Use subagents for research** — delegate exploratory work to subagents on cheap models to keep main context clean
- **Use `/compact`** for long sessions — e.g., `/compact keep only the implementation plan`
- **Skills load on demand** — move task-specific procedures into skills instead of bloating the agent body

### Model Selection Strategy

```yaml
# Orchestrator: strong reasoning for coordination
model: 'Claude Opus 4.5'

# Planner: deep understanding for architecture
model: ['Claude Opus 4.5', 'GPT-5.2']

# Implementer: good coding, cost-efficient
model: 'GPT-5 mini'           # 0× — free on paid plans

# Research subagent: fast, cheap, high volume
model: 'GPT-5 mini'           # 0× — unlimited research

# Security reviewer: needs deep reasoning (occasional use)
model: 'Claude Opus 4.6'      # 3× — justified for security
```

---

## 7. The Customization Stack — How Everything Fits Together

```
┌─────────────────────────────────────────────────────┐
│                  INSTRUCTIONS                        │
│  (Always-on rules: coding standards, conventions)    │
│                                                      │
│  copilot-instructions.md    *.instructions.md        │
│  AGENTS.md                  ~/.copilot/instructions  │
└──────────────────────┬──────────────────────────────┘
                       │ loaded on every request
                       ▼
┌─────────────────────────────────────────────────────┐
│                    AGENTS                            │
│  (Personas: who the model is, what it can do)        │
│                                                      │
│  *.agent.md (tools, model, handoffs, subagents)      │
│  Extension Chat Participants (TypeScript API)        │
└──────────────────────┬──────────────────────────────┘
                       │ selected by user or subagent invocation
                       ▼
┌─────────────────────────────────────────────────────┐
│                    SKILLS                            │
│  (Procedures: how to do specific tasks)              │
│                                                      │
│  SKILL.md (loaded on-demand via progressive loading) │
│  Scripts, templates, reference docs                  │
└──────────────────────┬──────────────────────────────┘
                       │ loaded when matched to task
                       ▼
┌─────────────────────────────────────────────────────┐
│                     TOOLS                            │
│  (Capabilities: what the agent can call)             │
│                                                      │
│  Built-in tools (edit, search, terminal)             │
│  MCP servers (external data, APIs)                   │
│  Extension-contributed tools (LM Tools API)          │
└──────────────────────┬──────────────────────────────┘
                       │ used during execution
                       ▼
┌─────────────────────────────────────────────────────┐
│                     HOOKS                            │
│  (Automation: deterministic actions at lifecycle     │
│   events — format, validate, block, audit)           │
│                                                      │
│  .github/hooks/*.json                                │
└─────────────────────────────────────────────────────┘
```

Each layer is independently upgradeable:
- Update standards without rewriting agents
- Upgrade skills without modifying instructions
- Change agent behavior while keeping conventions intact
- Everything lives in git — changes go through pull requests

---

## 8. Quick Decision Guide

```
WHAT DO YOU NEED TO BUILD?

├── A persona with specific tools/model for your team
│   └── .agent.md file in .github/agents/
│       (minutes to create, works everywhere)
│
├── An orchestrator that delegates to specialized workers
│   └── .agent.md with tools: ['agent'] + agents: [...]
│       (see Orchestration Guide)
│
├── A reusable workflow/procedure the agent can follow
│   └── Skill (SKILL.md) — not an agent
│       (see Skill Authoring Guide)
│
├── Deep IDE integration (debugger, custom UI, marketplace)
│   └── Extension Chat Participant (TypeScript)
│       (hours/days to build, VS Code only)
│
├── External data access (databases, APIs, services)
│   └── MCP server (connects to agents/skills via mcp-servers:)
│       (separate from agent authoring)
│
└── Deterministic automation (format on save, block commands)
    └── Hook (.github/hooks/*.json) — not an agent
        (see Customization Reference)
```

---

## 9. Key Resources

- **Custom Agents Docs:** [code.visualstudio.com/docs/copilot/customization/custom-agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents)
- **Chat Participant API:** [code.visualstudio.com/api/extension-guides/ai/chat](https://code.visualstudio.com/api/extension-guides/ai/chat)
- **Chat Extension Tutorial:** [code.visualstudio.com/api/extension-guides/ai/chat-tutorial](https://code.visualstudio.com/api/extension-guides/ai/chat-tutorial)
- **Chat Extension Utils:** [github.com/microsoft/vscode-chat-extension-utils](https://github.com/microsoft/vscode-chat-extension-utils)
- **Extension Samples:** [github.com/microsoft/vscode-extension-samples/chat-sample](https://github.com/microsoft/vscode-extension-samples/tree/main/chat-sample)
- **`@vscode/prompt-tsx`:** Token-budgeted prompt construction for extensions
- **Community Agents:** [github.com/github/awesome-copilot](https://github.com/github/awesome-copilot)
- **Agent Skills Spec:** [agentskills.io](https://agentskills.io)
