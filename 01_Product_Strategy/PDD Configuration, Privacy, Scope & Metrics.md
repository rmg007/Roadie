# PDD — Configuration, Privacy, Scope & Metrics

# Configuration Model

Roadie requires zero configuration. Every feature works out of the box with sensible defaults. Configuration exists solely as opt-in overrides for developers who want finer control. All configuration is stored in .vscode/settings.json under the roadie namespace.

| **Setting** | **Type** | **Default** | **Effect When Enabled** |
| --- | --- | --- | --- |
| roadie.telemetry | boolean | false | Sends anonymous, aggregate usage patterns. Never sends code, file names, or project details. |
| roadie.editTracking | boolean | false | Tracks developer edits to Roadie-generated files. Stores diffs locally. Used to learn preferences. |
| roadie.workflowHistory | boolean | false | Persists workflow outcomes. Used for pattern learning and self-optimization. |
| roadie.modelPreference | enum or null | null | Overrides the default model tier: "economy" (Tier 0 only), "balanced" (default), "quality" (start at Tier 1). |
| roadie.autoCommit | boolean | false | Stages and commits generated .github/ files with conventional commit messages. |
| roadie.testTimeout | number | 300 | Maximum seconds to wait for test suite execution. |
| roadie.testCommand | string or null | null | Overrides auto-detected test command for edge cases. |

> **Design principle:** Every boolean default is false. Roadie does nothing the developer hasn’t implicitly or explicitly consented to.
> 

---

# Privacy and Data Model

Roadie is designed around a local-first, privacy-preserving data model. No account is required, no cloud service is contacted, and the extension works fully offline.

## Data Storage

| **Data** | **Location** | **Gitignored** | **Retention** |
| --- | --- | --- | --- |
| Project model (SQLite) | .github/.roadie/project.db | Yes | Persists across sessions; rebuilt on demand |
| Section hashes | .github/.roadie/project.db | Yes | Updated each generation cycle |
| Edit tracking (opt-in) | .github/.roadie/project.db | Yes | Accumulated; no automatic pruning |
| Workflow history (opt-in) | .github/.roadie/project.db | Yes | Accumulated; no automatic pruning |
| Generated AI config files | .github/, [AGENTS.md](http://AGENTS.md) | No | Persist intentionally |
| Extension state | VS Code globalState | N/A | Managed by VS Code |

## What Is Never Stored

- Application source code (Roadie reads code but never copies it into its database)
- Chat conversation history (handled by VS Code’s chat infrastructure)
- Personal information, credentials, or secrets
- File contents beyond what the project model schema requires

## What Is Never Transmitted

- Code content, file names, or project details—even with telemetry enabled
- Any data to any server controlled by Roadie. Roadie has no server.
- LLM prompts go through the VS Code Language Model API to the developer’s Copilot subscription. Roadie does not proxy, log, or intercept these payloads.

---

# Scope Boundaries

## Roadie Does Not Replace

- **VS Code** — Roadie is an extension within VS Code, not an alternative.
- **GitHub Copilot** — Roadie enhances Copilot by providing better context and structured workflows.
- **Deterministic tools** — Formatters, type checkers, and test runners remain standalone. Roadie integrates them via hooks and shell commands.
- **Version control** — Roadie generates files but does not manage git operations beyond optional auto-commit.

## Roadie Does Not Do

- Run its own LLM. All model calls go through the VS Code Language Model API.
- Require an account, cloud service, or internet connection for core functionality.
- Modify application code outside of active workflow execution.
- Generate files eagerly at install time. All generation is lazy and event-driven.
- Handle monorepos in v1. If a monorepo is detected, Roadie operates at the root level.
- Target teams or enterprises. v1 is for solo developers.
- Intercept Copilot’s default chat agent invisibly. The developer explicitly selects Roadie.

## Agent-Based Tool Replacement

| **Replaced Tool** | **Roadie Equivalent** | **Key Difference** |
| --- | --- | --- |
| GitLeaks (secret scanning) | PreToolUse hooks + scanning skill | Detects secrets AND suggests remediation |
| npm audit / pip-audit | Dependency management workflow | Audits AND executes upgrades with test verification |
| Manual code review | Code review workflow (5-pass) | Multi-perspective review with project-specific rules |
| Manual doc auditing | Documentation workflow drift detection | Detects drift AND reconciles docs with code |

---

# Success Metrics

## Primary Metrics

| **Metric** | **Target** |
| --- | --- |
| Workflow completion rate (without developer intervention) | ≥85% within 30 days |
| First-attempt fix rate (Tier 0, no escalation) | ≥60% |
| Model cost efficiency (% calls at Tier 0) | ≥70% |
| Time-to-value (install → first workflow) | <5 minutes |
| Generated file accuracy (unedited within 7 days) | ≥90% |

## Anti-Metrics (What We Do NOT Optimize For)

- **Number of files generated** — Fewer, better files beat many unused ones.
- **LLM call volume** — Cost efficiency is a first-class design constraint.
- **User engagement frequency** — Success = less developer interruption, not more.
- **Feature count** — Depth of workflow quality > breadth of workflow types.