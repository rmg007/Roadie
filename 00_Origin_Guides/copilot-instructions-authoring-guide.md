# Writing Copilot Instructions — Practical Authoring Guide
**Updated:** April 2026 · **Covers:** VS Code 1.110+, Copilot CLI, Copilot Cloud Agent, Code Review  
**Companion guides:** Customization Reference, Skill Authoring Guide, Agent Authoring Guide

---

## 1. What Instructions Are (and Aren't)

Instructions are **always-on rules** that Copilot includes in every request within their scope. They define *how* the AI should write code in your project — coding standards, architectural patterns, conventions, and constraints.

| Instructions are for... | Instructions are NOT for... |
| :--- | :--- |
| Coding standards and conventions | Task-specific procedures → use **Skills** |
| Architectural patterns and decisions | Persistent personas with tool restrictions → use **Agents** |
| Tech stack and framework rules | One-off reusable prompts → use **Prompt files** |
| Security policies and constraints | Things linters/formatters already enforce |
| Non-obvious project-specific patterns | Generic advice ("write clean code", "handle errors") |
| Build/test/deploy commands | Comprehensive documentation |

**Key cost:** Instructions load on **every** chat interaction. Every line costs tokens. Keep them lean — only include what the model can't infer from reading your code.

---

## 2. Instruction Types & Priority Order

### Types (from most specific to most general)

| Type | File | Scope | Loaded When |
| :--- | :--- | :--- | :--- |
| **Personal** | VS Code user settings or user-level instruction files | Your machine, all repos | Every request |
| **Path-specific** | `.github/instructions/*.instructions.md` | Files matching `applyTo` glob | When editing matching files |
| **Repository-wide** | `.github/copilot-instructions.md` | Entire workspace | Every request |
| **Cross-agent** | `AGENTS.md` (root) | Entire workspace, all agents | Every request |
| **Organization** | GitHub org settings | All repos in org | Every request |

### Priority Order (highest → lowest)

When instructions conflict, higher-priority wins:

```
1. Personal instructions         (your user settings)
2. Path-specific instructions    (.github/instructions/*.instructions.md)
3. Repository-wide instructions  (.github/copilot-instructions.md)
4. AGENTS.md                     (root)
5. Organization-level            (GitHub org settings)
```

### Which File to Use

```
Is the rule UNIVERSAL across all your projects?
  → Personal instructions (user-level)

Is the rule specific to THIS REPO and applies to ALL files?
  → .github/copilot-instructions.md

Is the rule specific to a LANGUAGE, FRAMEWORK, or FOLDER?
  → .github/instructions/{topic}.instructions.md with applyTo glob

Should the rule apply to ALL AI agents (Copilot + Claude + Gemini)?
  → AGENTS.md in the repo root

Should the rule apply across your entire ORGANIZATION?
  → GitHub organization-level instructions
```

---

## 3. Writing `copilot-instructions.md` (Repository-Wide)

This is the primary file. It's included in **every** Copilot request in the workspace. No frontmatter — pure Markdown.

### Quick Start

Type `/init` in chat to auto-generate one tailored to your codebase. Or type `/create-instructions` with a description.

### What to Include (The Five Essential Sections)

Based on GitHub's official guidance:

**1. Project Overview** — What this project does, in 2-3 sentences.

**2. Tech Stack** — Languages, frameworks, databases, and their versions. This prevents the model from guessing or suggesting outdated alternatives.

**3. Coding Standards** — Non-obvious rules the model can't infer from code. Skip anything a linter enforces.

**4. Project Structure** — Where things go. Feature folders, naming patterns, file organization.

**5. Build/Test/Run Commands** — How to build, test, and run the project. Critical for agent mode.

### Complete Example

```markdown
# Project Overview
Recipe sharing web app. Users create, browse, and rate recipes.

## Tech Stack
- TypeScript 5.x (strict mode)
- Next.js 14 (App Router, Server Components by default)
- PostgreSQL via Prisma ORM
- Tailwind CSS v4
- Vitest (unit) + Playwright (E2E)

## Coding Standards
- Named exports only — no default exports
- All async functions must have try/catch with structured logging
- Use `date-fns` instead of `moment.js` (moment is deprecated, increases bundle size)
- Server Components by default; add `"use client"` only when hooks/events are needed
- Props interfaces named `{ComponentName}Props`, defined above the component

## Project Structure
- `src/features/{name}/` — feature folders with components, hooks, utils
- `src/components/ui/` — shared design system components
- `src/lib/` — framework-agnostic utilities
- `tests/e2e/` — Playwright tests
- `prisma/` — schema and migrations

## Build & Run
- `pnpm install` → `pnpm dev` → http://localhost:3000
- Unit tests: `pnpm test`
- E2E tests: `pnpm test:e2e`
- Lint: `pnpm lint`
- DB migrations: `pnpm prisma migrate dev`

## PR Standards
- Conventional commits: feat|fix|chore|docs|refactor|test
- PR title must reference an issue number
- All PRs must pass CI before merge
```

### Writing Rules — Do's and Don'ts

**✅ Do:**

- **Keep it short** — aim for 20–50 lines. Under 100 max.
- **Be specific to THIS project** — not generic advice.
- **Include the "why"** — *"Use date-fns instead of moment.js because moment is deprecated and increases bundle size"* helps the model make better edge-case decisions.
- **Show examples** — the model follows concrete code examples better than abstract rules:

```markdown
## Import Order
1. External libraries
2. Internal modules (`@/`)
3. Relative paths (`./`)

Example:
```typescript
import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { formatDate } from './utils';
```
```

- **State what NOT to do** — explicit anti-patterns prevent common mistakes:

```markdown
## Avoid
- Do NOT use `any` type — use `unknown` and narrow
- Do NOT write raw SQL — use Prisma client
- Do NOT use `console.log` — use the structured logger (`src/lib/logger`)
```

**❌ Don't:**

- Don't include things linters enforce (semicolons, indentation, trailing commas)
- Don't write generic advice ("write clean code", "use meaningful names")
- Don't document aspirational practices — only things actually used in the codebase
- Don't exceed ~100 lines — use path-specific instructions for language/framework details
- Don't include task-specific procedures — move those to **Skills**

---

## 4. Writing Path-Specific Instructions (`*.instructions.md`)

For rules that only apply to specific file types or directories. Each file uses `applyTo` frontmatter with a glob pattern.

### Frontmatter

```yaml
---
applyTo: "**/*.tsx,**/*.jsx"        # Required: glob pattern(s), comma-separated
description: "React component rules" # Optional: human-readable description
excludeAgent: "copilot-code-review"  # Optional: exclude from specific agents
---
```

### When to Use

- **Language-specific rules** — Python conventions that don't apply to TypeScript
- **Framework-specific rules** — React patterns that don't apply to backend code
- **Directory-specific rules** — different standards for `/tests/` vs `/src/`
- **Agent-specific exclusion** — rules that should apply to coding but not code review (or vice versa)

### Examples

**`react.instructions.md`**
```markdown
---
applyTo: "**/*.tsx,**/*.jsx"
---

## React Rules
- Functional components only — no class components
- Props interface named `{ComponentName}Props`, defined above the component
- Server Components by default; add `"use client"` only for hooks/events
- State: `useState` for local, TanStack Query for server state
- Never fetch data in `useEffect` — use TanStack Query or Server Components

## Component Structure
```tsx
// 1. Imports
// 2. Types/interfaces
// 3. Component function
// 4. Helper functions (if any)
// 5. Export
```
```

**`python.instructions.md`**
```markdown
---
applyTo: "**/*.py"
---

## Python Rules
- Python 3.12+ with type hints on all function signatures
- Use `pydantic` for data validation, not raw dicts
- Use `ruff` for formatting (already configured in pyproject.toml)
- Prefer `pathlib.Path` over `os.path`
- Use `httpx` for HTTP requests (not `requests`)
```

**`testing.instructions.md`**
```markdown
---
applyTo: "tests/**,**/*.test.*,**/*.spec.*"
---

## Testing Rules
- Use AAA pattern: Arrange-Act-Assert
- Test file naming: `{module}.test.ts`
- Use `describe` blocks for grouping, `it` blocks for individual cases
- Always test: happy path, error path, boundary values
- Use fixtures from `tests/fixtures/` — don't hardcode test data
- Mock external services — never hit real APIs in tests
```

**`api.instructions.md` (exclude from code review)**
```markdown
---
applyTo: "src/api/**"
excludeAgent: "copilot-code-review"
---

## API Development Notes
These rules are for coding assistance only (not code review):
- All endpoints must validate input with Zod schemas
- Use the `ApiResponse<T>` wrapper for all responses
- Rate limiting is handled by middleware — don't add it per-endpoint
```

---

## 5. Writing `AGENTS.md` (Cross-Agent)

Use this when you work with multiple AI tools (Copilot, Claude Code, Gemini CLI, etc.) and want shared rules. It's recognized by the open agent standard.

```markdown
# Project: RecipeApp

## Overview
Recipe sharing web app built with Next.js 14, TypeScript, Prisma, Tailwind.

## Universal Rules
- TypeScript strict mode — no `any`, no implicit returns
- Named exports only
- Tests required for all public APIs
- Use the structured logger, never console.log

## Out of Bounds
- NEVER commit secrets or API keys
- NEVER modify existing migration files
- NEVER use `eval()` or dynamic code execution
```

If you have both `AGENTS.md` and `copilot-instructions.md`, both are loaded. `copilot-instructions.md` takes priority on conflicts.

---

## 6. Personal Instructions (User-Level)

Rules that apply to all your projects, everywhere.

### VS Code Settings

For code review, commit messages, and PR descriptions, use settings:

```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "text": "Always use const over let unless mutation is required" },
    { "file": "~/my-instructions.md" }
  ]
}
```

> **Note:** Settings-based code generation and test generation instructions are **deprecated as of VS Code 1.102**. Use file-based instructions instead.

### File-Based Personal Instructions

Create instruction files in your user profile folder. Sync across machines with Settings Sync → select *Prompts and Instructions*.

---

## 7. Organization-Level Instructions

Define instructions at the GitHub organization level to apply across all repos. Enable discovery with: `github.copilot.chat.organizationCustomAgents.enabled`.

These appear in the Chat Instructions menu alongside personal and workspace instructions. They have the **lowest priority** — repo-level instructions override them.

---

## 8. Common Patterns

### Pattern 1: Tech Stack Declaration

```markdown
## Tech Stack
- Language: TypeScript 5.x (strict mode)
- Framework: Next.js 14 (App Router)
- Database: PostgreSQL via Prisma ORM 6.x
- Styling: Tailwind CSS v4
- Auth: NextAuth.js v5
- Testing: Vitest + Playwright
- Package manager: pnpm
```

*Why it works:* Prevents the model from suggesting Express when you use Next.js, or Sequelize when you use Prisma.

### Pattern 2: Architecture Decision Records

```markdown
## Architecture Decisions
- **State management:** Use TanStack Query for server state, Zustand for client
  state. Do NOT use Redux (removed in v3.0 migration).
- **API layer:** All external API calls go through `src/lib/api-client.ts`.
  Never call `fetch()` directly from components.
- **Feature flags:** Use LaunchDarkly SDK. Check `src/lib/feature-flags.ts`
  for the wrapper. Never hardcode feature checks.
```

*Why it works:* Explains decisions the model can't infer from code alone.

### Pattern 3: Error Handling Convention

```markdown
## Error Handling
All errors must use the `AppError` class from `src/errors/`:

```typescript
// ✅ Correct
throw new AppError('USER_NOT_FOUND', 'User does not exist', 404);

// ❌ Wrong — never throw raw Error
throw new Error('User not found');
```

Log with structured logger:
```typescript
logger.error({ code: 'USER_NOT_FOUND', userId, context: 'getUser' });
```
```

*Why it works:* Concrete examples beat abstract rules.

### Pattern 4: What NOT to Do

```markdown
## Banned Patterns
- Do NOT use `any` — use `unknown` and narrow with type guards
- Do NOT use `moment.js` — use `date-fns` (moment is deprecated)
- Do NOT write raw SQL — use Prisma client
- Do NOT use class components — use functional components with hooks
- Do NOT commit `.env` files — use `.env.example` with placeholder values
```

*Why it works:* Explicit anti-patterns prevent the model's tendency to suggest popular-but-wrong patterns.

---

## 9. Anti-Patterns (What NOT to Put in Instructions)

| Anti-Pattern | Why It's Bad | Fix |
| :--- | :--- | :--- |
| Generic advice ("write clean code") | Wastes tokens, model already tries this | Be specific to your project |
| Things linters enforce (semicolons, indent) | Redundant — linter catches these anyway | Only include rules linters can't enforce |
| Aspirational practices not in the codebase | Model will suggest patterns that don't match existing code | Only document what's actually used |
| Task-specific procedures ("how to deploy") | Bloats always-on context for rarely-used info | Move to a **Skill** |
| Entire API documentation | Massive token cost, loaded on every request | Reference via MCP server or link |
| Role/persona definitions | Instructions don't change *who* the model is | Use a **Custom Agent** |
| Copy of README.md | Redundant — model can read the file when needed | Reference it instead |

---

## 10. The Iterative Process

Instructions are not write-once. They improve through observation:

```
1. START MINIMAL
   - Use /init to generate a baseline
   - Or write 20-30 lines covering the five essential sections

2. OBSERVE FAILURES
   - When Copilot suggests wrong patterns, note what it got wrong
   - When Copilot asks questions it shouldn't need to, add the answer

3. ADD TARGETED RULES
   - Add one rule per observed failure
   - Include the "why" so the model generalizes correctly
   - Add a concrete example (positive and/or negative)

4. SPLIT WHEN NEEDED
   - If copilot-instructions.md exceeds ~50 lines, move
     language/framework-specific rules to *.instructions.md files
   - If task-specific procedures accumulate, move them to Skills

5. REVIEW PERIODICALLY
   - Remove rules that are no longer relevant
   - Update tech stack versions
   - Check if rules are actually being followed
```

---

## 11. Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════╗
║  INSTRUCTION FILE CHECKLIST                                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  .github/copilot-instructions.md (repo-wide, always-on):        ║
║    □ Project overview (2-3 sentences)                            ║
║    □ Tech stack with versions                                    ║
║    □ Non-obvious coding standards (skip linter-enforced rules)   ║
║    □ Project structure (where things go)                         ║
║    □ Build/test/run commands                                     ║
║    □ Under 100 lines (ideally 20-50)                             ║
║    □ No frontmatter — pure Markdown                              ║
║                                                                  ║
║  .github/instructions/*.instructions.md (path-specific):         ║
║    □ applyTo glob in YAML frontmatter                            ║
║    □ Language or framework-specific rules only                   ║
║    □ Scoped to avoid context pollution                           ║
║                                                                  ║
║  WRITING RULES:                                                  ║
║    □ Be specific to THIS project                                 ║
║    □ Include the "why" behind each rule                          ║
║    □ Show concrete code examples (positive + negative)           ║
║    □ State anti-patterns explicitly                              ║
║    □ Only rules the model can't infer from code                  ║
║    □ Skip generic advice and linter-enforced rules               ║
║                                                                  ║
║  QUICK START:                                                    ║
║    /init ................ auto-generate from codebase             ║
║    /create-instructions . generate from description               ║
║                                                                  ║
║  PRIORITY (highest → lowest):                                    ║
║    Personal → Path-specific → Repo-wide → AGENTS.md → Org       ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 12. Key Resources

- **VS Code Instructions Docs:** [code.visualstudio.com/docs/copilot/customization/custom-instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- **GitHub Blog — 5 Tips:** [github.blog/ai-and-ml/github-copilot/5-tips-for-writing-better-custom-instructions-for-copilot](https://github.blog/ai-and-ml/github-copilot/5-tips-for-writing-better-custom-instructions-for-copilot/)
- **Code Review Instructions:** [github.blog/ai-and-ml/unlocking-the-full-power-of-copilot-code-review-master-your-instructions-files](https://github.blog/ai-and-ml/unlocking-the-full-power-of-copilot-code-review-master-your-instructions-files/)
- **GitHub Tutorial:** [docs.github.com/copilot/tutorials/use-custom-instructions](https://docs.github.com/en/copilot/tutorials/use-custom-instructions)
- **Best Practices:** [code.visualstudio.com/docs/copilot/best-practices](https://code.visualstudio.com/docs/copilot/best-practices)
- **Community Examples:** [github.com/github/awesome-copilot](https://github.com/github/awesome-copilot)
