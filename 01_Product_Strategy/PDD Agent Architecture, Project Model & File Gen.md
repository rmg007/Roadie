# PDD — Agent Architecture, Project Model & File Generation

Roadie uses the term “agent” to describe role-specific prompt configurations, not separate VS Code Chat Participants. There is one Chat Participant—Roadie itself—registered via the VS Code Chat Participant API. All internal “agents” are an abstraction: a combination of a system prompt template, a scoped tool set, and a model-tier preference. The developer never sees or interacts with individual agents directly.

## The Chat Participant: Roadie

The developer selects Roadie from the VS Code chat agent dropdown (the same dropdown where @workspace and @terminal appear). Once selected, all prompts flow through Roadie’s Chat Participant handler. The handler has two responsibilities: intent classification (routing to workflows) and passthrough (forwarding non-workflow prompts to the underlying model with enhanced context from the project model).

> Roadie does NOT invisibly intercept Copilot’s default agent. The developer makes an explicit selection once, then operates naturally. The invisibility is in the execution, not the invocation.
> 

## Intent Classifier

The intent classifier uses a two-tier approach. Tier 1 is a local keyword/regex classifier that runs instantly with zero cost. If local confidence is below 0.7, Tier 2 uses the first LLM call (piggybacked on the response, not a separate call) to classify with structured output. Classification latency target: under 500ms (under 10ms for local tier).

If the prompt does not match any workflow (general questions, casual chat, code explanations), Roadie passes the prompt to the underlying model with the project model injected as context. This means even non-workflow prompts benefit from Roadie’s project awareness.

## Internal Agent Roles

| **Agent** | **Role** | **Tools** | **Tier** | **Lifecycle** |
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

All agents are ephemeral (created for a workflow, destroyed after) except the Project Analyzer, which persists across the extension’s lifecycle.

## Agent Implementation

Each agent is implemented as a TypeScript class with: `systemPrompt` (template string with project context placeholders), `tools` (whitelist of tool identifiers), `modelPreference` (starting and maximum tier), and `validate(output)` (quality validation function that triggers escalation on failure).