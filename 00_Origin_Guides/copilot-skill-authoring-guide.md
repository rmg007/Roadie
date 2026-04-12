# Writing Agent Skills for GitHub Copilot — Practical Authoring Guide
**Updated:** April 2026 · **Covers:** VS Code 1.110+, Copilot CLI, Copilot Cloud Agent  
**Spec:** [agentskills.io](https://agentskills.io) open standard (30+ compatible tools)  
**Purpose:** Everything an AI coding agent (or developer) needs to create effective, secure, portable skills.

---

## 1. What a Skill Is (and Isn't)

A skill is a **self-contained folder** of instructions, scripts, and resources that Copilot loads **on demand** when relevant to the current task. Unlike custom instructions (always-on, every request), skills use progressive disclosure — only loading when matched.

| Use a Skill when... | Use Custom Instructions when... | Use a Custom Agent when... |
| :--- | :--- | :--- |
| Task-specific workflow (testing, deployment, migration) | Repo-wide coding standards (naming, style, patterns) | Persistent persona with tool restrictions |
| Includes scripts, templates, or reference files | Short rules that apply to everything | Orchestrates multiple skills via subagents |
| Should only load when relevant (saves context) | Must always be in context | Needs handoffs between roles |
| Needs to be portable across tools (Copilot, Claude Code, Codex, Cursor, Gemini CLI) | Only needed in this repo's Copilot | Needs model or MCP server scoping |

**Rule of thumb:** If you find yourself copy-pasting the same instructions into prompts repeatedly, it should be a skill. If a section of your `copilot-instructions.md` only matters for specific tasks, move it to a skill.

---

## 2. How Progressive Loading Works (3 Levels)

Understanding this is critical for writing effective skills:

```
Level 1 — DISCOVERY (always loaded, very cheap)
  Only the `name` and `description` from frontmatter are loaded into
  the system prompt. This is how Copilot decides if your skill is relevant.
  → Your description MUST be excellent. This is the only thing Copilot
    reads when deciding whether to load your skill.

Level 2 — INSTRUCTIONS (loaded on match)
  When Copilot matches your skill to the user's task (via description
  or /slash command), the full SKILL.md body is injected into context.
  → Keep the body focused and action-oriented. Every word costs tokens.

Level 3 — RESOURCES (loaded on reference)
  As Copilot works through your instructions, it accesses files in the
  skill directory ONLY when they are referenced via relative links.
  → Don't dump everything in SKILL.md. Put details in separate files
    and link to them. They load only when needed.
```

**Implication:** You can install dozens of skills with near-zero context cost. Only relevant skills load, and only referenced resources within those skills get injected.

---

## 3. Directory Structure & Naming Rules

### Strict Rules

- Each skill lives in its **own subdirectory**
- Directory name: **lowercase, hyphens only**, max 64 characters
- `name` in frontmatter **must exactly match** directory name (or skill won't load)
- File must be named exactly `SKILL.md` (case-sensitive)

### Supported Locations

| Location | Scope | When to Use |
| :--- | :--- | :--- |
| `.github/skills/{name}/` | Repository | Team standards, project-specific workflows |
| `.claude/skills/{name}/` | Repository (Claude Code compat) | Cross-tool compatibility |
| `.agents/skills/{name}/` | Repository (generic) | Maximum portability across all tools |
| `~/.copilot/skills/{name}/` | Personal (all repos) | Personal preferences, environment-specific |
| `~/.claude/skills/{name}/` | Personal (Claude Code compat) | Personal, cross-tool |
| `~/.agents/skills/{name}/` | Personal (generic) | Personal, maximum portability |

> **For monorepos:** Enable `chat.useCustomizationsInParentRepositories` to discover skills from parent directories.

### Example Structure

```text
.github/skills/
├── webapp-testing/
│   ├── SKILL.md              # Required — definition + instructions
│   ├── test-template.ts       # Template Copilot can use
│   ├── examples/
│   │   ├── login-test.ts      # Example loaded only when referenced
│   │   └── api-test.ts
│   └── scripts/
│       └── run-coverage.sh    # Script the agent can execute
│
├── api-scaffolding/
│   ├── SKILL.md
│   └── templates/
│       ├── controller.ts.tmpl
│       ├── service.ts.tmpl
│       └── dto.ts.tmpl
│
└── db-migration/
    ├── SKILL.md
    └── migration-checklist.md  # Reference doc loaded on demand
```

---

## 4. SKILL.md Anatomy — Complete Reference

### Frontmatter Fields

```yaml
---
# === Required ===
name: webapp-testing              # Must match directory name. Lowercase + hyphens. Max 64 chars.
description: >-                   # Max 1024 chars. THE most important field — see Section 5.
  Guide for testing web applications using Playwright.
  Use when asked to create, fix, or run browser-based tests,
  write E2E tests, set up test infrastructure, or debug test failures.

# === Optional: Invocation Control ===
argument-hint: "[file or module]"  # Hint shown in chat input when invoked via /slash command
user-invocable: true               # Default: true. Set false to hide from / menu (agent-only skill)
disable-model-invocation: false    # Default: false. Set true = must use /slash, no auto-matching

# === Optional: Security ===
allowed-tools:                     # Tools Copilot can use WITHOUT asking for confirmation
  - terminal                       # ⚠️ Only for trusted, self-authored skills. See Section 7.

# === Optional: Metadata ===
license: MIT
---
```

### Body Structure

The body is standard Markdown. Write it as **instructions for an AI agent**, not documentation for humans.

```markdown
# Web Application Testing with Playwright

## When to Use This Skill
- Creating new browser-based tests
- Debugging failing Playwright tests
- Setting up test infrastructure from scratch
- Adding test coverage to existing features

## Prerequisites
- Playwright is installed (`npx playwright install`)
- Test directory exists at `tests/e2e/`

## Procedure
1. Identify the feature or page to test
2. Check for existing tests in `tests/e2e/` — avoid duplicates
3. Create a new test file using the [test template](./test-template.ts)
4. Follow the AAA pattern (Arrange-Act-Assert)
5. Include these test categories:
   - Happy path (expected behavior)
   - Error path (invalid input, network failures)
   - Boundary values (empty strings, max lengths, special characters)
6. Run tests: `npx playwright test --headed` for visual, `npx playwright test` for CI
7. Verify all pass before committing

## Naming Convention
- File: `{feature}.test.ts` (e.g., `login.test.ts`)
- Describe blocks: feature name
- It blocks: "should {expected behavior} when {condition}"

## Example
See [login test example](./examples/login-test.ts) for a complete reference.

## Common Pitfalls
- Do NOT use `page.waitForTimeout()` — use `page.waitForSelector()` instead
- Do NOT hardcode test data — use fixtures from `tests/fixtures/`
- Always clean up test state in `afterEach`
```

---

## 5. Writing Effective Descriptions (The Most Important Skill)

The `description` field is how Copilot decides whether to load your skill. A bad description means your skill never activates. A vague description means it activates when it shouldn't, wasting context.

### Rules for Descriptions

1. **State what the skill does** (capabilities)
2. **State when to use it** (trigger conditions)
3. **Include keywords** the user would naturally type
4. **Max 1024 characters** — use them wisely
5. **Be specific** — vague descriptions cause false matches or missed matches

### Good vs Bad Descriptions

**❌ Bad — too vague:**
```yaml
description: Helps with testing
```
*Problem: Matches everything remotely related to "testing." Will load for unit tests, integration tests, manual testing discussions — drowning the context.*

**❌ Bad — too narrow:**
```yaml
description: Runs Playwright tests on the login page
```
*Problem: Won't match "help me test the checkout flow" or "set up E2E infrastructure."*

**✅ Good — specific capabilities + trigger keywords:**
```yaml
description: >-
  Guide for testing web applications using Playwright.
  Use when asked to create, fix, or run browser-based tests,
  write E2E tests, set up test infrastructure, or debug test failures.
  Covers page objects, fixtures, visual regression, and CI integration.
```

**✅ Good — action verbs + domain terms:**
```yaml
description: >-
  Scaffolds new REST API endpoints following the project's Clean Architecture.
  Use when asked to create a new endpoint, add a controller, generate DTOs,
  or scaffold CRUD operations. Includes templates for controllers, services,
  repositories, and validation.
```

**✅ Good — clear boundaries:**
```yaml
description: >-
  Manages database migrations using Prisma. Use when asked to create, run,
  or troubleshoot migrations, update the schema, seed the database, or
  resolve migration conflicts. Do NOT use for raw SQL queries or ORM
  usage outside of migrations.
```

### Description Design Pattern

```
[Action verb] [what it does] using [technology/framework].
Use when [trigger condition 1], [trigger condition 2], or [trigger condition 3].
Covers [specific capability 1], [capability 2], and [capability 3].
```

---

## 6. Skill Patterns Catalog

### Pattern 1: Procedural Skill (step-by-step workflow)

The most common pattern. Guides the agent through a multi-step process.

```markdown
---
name: pr-review-checklist
description: >-
  Guides code review of pull requests against team standards.
  Use when reviewing a PR, checking code quality, or performing
  pre-merge validation.
---

# PR Review Checklist

## Procedure
1. Check that all files follow naming conventions
2. Verify test coverage for changed files (run `pnpm test --coverage`)
3. Check for:
   - Hardcoded strings that should be i18n keys
   - Console.log statements (should use structured logger)
   - Missing error handling in async functions
   - Unused imports
4. Verify API changes have corresponding OpenAPI spec updates
5. Confirm migration files (if any) are reversible
6. Output a structured review with severity levels (Critical/Warning/Info)
```

### Pattern 2: Template Skill (generates from templates)

Includes template files the agent fills in based on the user's request.

```text
.github/skills/api-endpoint/
├── SKILL.md
└── templates/
    ├── controller.ts.tmpl
    ├── service.ts.tmpl
    ├── dto.ts.tmpl
    └── test.ts.tmpl
```

```markdown
---
name: api-endpoint
description: >-
  Scaffolds a new REST API endpoint with controller, service, DTO, and tests.
  Use when asked to create a new endpoint, add CRUD operations, or scaffold
  an API resource.
argument-hint: "resource name (e.g., 'users', 'products')"
---

# API Endpoint Scaffolding

## Procedure
1. Ask for the resource name if not provided
2. Create files using these templates:
   - Controller: [controller.ts.tmpl](./templates/controller.ts.tmpl)
   - Service: [service.ts.tmpl](./templates/service.ts.tmpl)
   - DTO: [dto.ts.tmpl](./templates/dto.ts.tmpl)
   - Tests: [test.ts.tmpl](./templates/test.ts.tmpl)
3. Place files in the correct directories:
   - `src/controllers/{resource}.controller.ts`
   - `src/services/{resource}.service.ts`
   - `src/dto/{resource}.dto.ts`
   - `tests/{resource}.test.ts`
4. Register the controller in `src/app.module.ts`
5. Add route to `src/routes/index.ts`
```

### Pattern 3: Script Skill (runs scripts)

Wraps shell scripts the agent can execute.

```text
.github/skills/image-convert/
├── SKILL.md
└── convert-svg-to-png.sh
```

```markdown
---
name: image-convert
description: >-
  Converts SVG images to PNG format with configurable dimensions.
  Use when asked to convert SVG to PNG, resize images, or generate
  image assets from vector files.
allowed-tools: [terminal]
---

# Image Conversion

When asked to convert an SVG to PNG, run the `convert-svg-to-png.sh` script
from this skill's base directory:

```bash
./convert-svg-to-png.sh <input-svg-path> [width] [height]
```

- Default dimensions: 1024×1024
- Output is saved alongside the input file with `.png` extension
- Requires `librsvg` (install via `brew install librsvg` or `apt install librsvg2-bin`)
```

### Pattern 4: Reference Skill (knowledge base)

No scripts — just structured knowledge the agent can consult.

```text
.github/skills/error-codes/
├── SKILL.md
└── references/
    ├── api-errors.md
    ├── database-errors.md
    └── auth-errors.md
```

```markdown
---
name: error-codes
description: >-
  Reference for project error codes and their meanings.
  Use when debugging errors, interpreting error codes, or implementing
  error handling. Covers API errors, database errors, and auth errors.
---

# Error Code Reference

## Quick Lookup
- API errors (4xx/5xx): see [API error reference](./references/api-errors.md)
- Database errors (DB-xxx): see [database error reference](./references/database-errors.md)
- Auth errors (AUTH-xxx): see [auth error reference](./references/auth-errors.md)

## Error Handling Pattern
All errors must follow this structure:
- Use the `AppError` class from `src/errors/`
- Include error code, human-readable message, and HTTP status
- Log with structured logger: `logger.error({ code, message, context })`
- Never expose internal error details to API consumers
```

### Pattern 5: Validation Skill (checks/audits)

Runs checks and reports findings.

```markdown
---
name: security-audit
description: >-
  Scans code for common security vulnerabilities using OWASP guidelines.
  Use when asked to review security, check for vulnerabilities, audit
  authentication, or scan for hardcoded secrets.
---

# Security Audit

## Procedure
1. Scan for hardcoded secrets:
   - Search for patterns: API keys, tokens, passwords, connection strings
   - Check `.env` files are in `.gitignore`
   - Look for secrets in test fixtures

2. Check authentication:
   - All endpoints have auth middleware
   - Tokens have expiration
   - Password hashing uses bcrypt with ≥10 rounds

3. Check injection vulnerabilities:
   - SQL: verify parameterized queries (no string concatenation)
   - XSS: verify output encoding
   - Command injection: verify input sanitization

4. Check dependencies:
   - Run `npm audit` or `pnpm audit`
   - Flag any high/critical CVEs

## Output Format
For each finding:
- **Severity:** Critical / High / Medium / Low
- **Location:** file:line
- **Issue:** what's wrong
- **Fix:** specific remediation
```

---

## 7. Security

### The `allowed-tools` Risk

By default, Copilot asks for human confirmation before running terminal commands. The `allowed-tools` field **bypasses this prompt** for listed tools.

```yaml
# ⚠️ This means Copilot can run shell commands WITHOUT asking you
allowed-tools: [terminal]
```

**Rules:**
- **Only use `allowed-tools` for skills you wrote yourself** and fully trust
- **Never** on skills downloaded from the internet without review
- If the skill processes **untrusted input** (user uploads, external data, API responses), **do not** auto-approve tools
- When in doubt, **omit `allowed-tools`** — the confirmation prompt is your safety net

### Script Safety

When writing scripts that skills will execute:
- **Be explicit** about arguments: *"Run `./run-tests.sh <file-path>` with exactly one argument"*
- **Validate inputs** in the script itself (check file exists, sanitize paths)
- **Never** pass raw user input directly to `eval`, `exec`, or shell interpolation
- **Restrict file operations** to the project directory (no `../../` traversal)

### Review Shared Skills

Before adding a community skill to your project:
1. Read the entire SKILL.md — understand what it does
2. Read all scripts — check for unexpected network calls, file operations, or data exfiltration
3. Check `allowed-tools` — if it auto-approves terminal, scrutinize extra carefully
4. Open untrusted repos in **Restricted Mode** first

---

## 8. Invocation Control

| Frontmatter | In `/` menu? | Auto-matched? | Use Case |
| :--- | :--- | :--- | :--- |
| *(defaults)* | ✅ | ✅ | Normal skill — available everywhere |
| `user-invocable: false` | ❌ | ✅ | Agent-only skill — no slash command |
| `disable-model-invocation: true` | ✅ | ❌ | Manual-only — must type `/name` |
| Both `false`/`true` | ❌ | ❌ | Effectively disabled |

### When to Use Each

- **Default (both true):** Most skills. Let Copilot match automatically AND let users invoke manually.
- **`user-invocable: false`:** Internal skills that agents should use but users shouldn't invoke directly (e.g., a research skill only called by an orchestrator subagent).
- **`disable-model-invocation: true`:** Destructive or expensive skills that should only run when the user explicitly asks (e.g., a database migration skill).

---

## 9. Creating Skills Efficiently

### From Chat

```
/create-skill a skill for running and debugging integration tests
```

Copilot asks clarifying questions and generates the SKILL.md, directory structure, and frontmatter.

### From Conversation

After a multi-turn session where you solved a complex problem:

```
Create a skill from how we just debugged that authentication issue
```

Copilot extracts the multi-step procedure into a reusable skill.

### From the Customizations Editor

1. Command Palette → `Chat: Open Chat Customizations`
2. Select the **Skills** tab
3. Click **New Skill (Workspace)** or **New Skill (User)**
4. Enter a name and description
5. Fill in the generated SKILL.md

### Verifying Skills

- Type `/skills` in chat to open the Configure Skills menu
- Type `/` to see available skills in the slash command list
- Ask Copilot: *"What skills do you have available?"*

---

## 10. Writing Tips for Skill Bodies

### Do

- **Write instructions for an AI agent**, not documentation for humans
- **Be imperative:** "Run the test. Check the output. Fix failures." Not "The test can be run by..."
- **Reference files with relative links:** `[template](./template.ts)` — this triggers Level 3 loading
- **Include examples of expected output** — agents follow examples better than abstract rules
- **State what NOT to do** — explicit anti-patterns prevent common mistakes
- **Keep it concise** — every token in the body costs context when the skill is loaded

### Don't

- **Don't duplicate copilot-instructions.md** — the skill will be loaded alongside it
- **Don't include general coding advice** — focus on what's unique to this skill's task
- **Don't put everything in SKILL.md** — use separate files linked via relative paths for details
- **Don't write vague instructions** — "make it good" doesn't help an agent; "follow the AAA pattern with describe/it blocks" does
- **Don't forget the prerequisites** — state required tools, packages, or setup steps

### Token Budget

- **Description:** Max 1024 characters (Level 1 — always loaded, keep tight)
- **SKILL.md body:** No hard limit, but aim for **under 500 lines**. Longer skills should split details into referenced files.
- **Referenced files:** Loaded only when the agent accesses them. Can be longer.

---

## 11. Skills vs Instructions vs Agents — Decision Matrix

```
Is this a RULE that applies to all code in this repo?
  → copilot-instructions.md (or *.instructions.md with applyTo)
  Examples: "use named exports," "no any type," "conventional commits"

Is this a PROCEDURE for a specific task?
  → Skill (SKILL.md)
  Examples: "how to write E2E tests," "how to deploy to staging,"
            "how to scaffold a new API endpoint"

Is this a PERSONA with tool restrictions and handoffs?
  → Custom Agent (*.agent.md)
  Examples: "read-only code reviewer," "planning-only architect,"
            "orchestrator that delegates to subagents"

Does the persona need to USE skills?
  → Agent that invokes skills via its instructions
  Example: "Use the /security-audit skill to check changed files,
           then use the /pr-review-checklist skill to review quality."
```

---

## 12. Distribution & Plugins

### Package Skills into Plugins

Bundle related skills, agents, and instructions into a single installable plugin:

```text
my-team-standards/
├── plugin.json           # Plugin manifest
├── skills/
│   ├── testing/
│   │   └── SKILL.md
│   ├── api-scaffold/
│   │   └── SKILL.md
│   └── deployment/
│       └── SKILL.md
├── agents/
│   └── reviewer.agent.md
└── instructions/
    └── react.instructions.md
```

### Install from CLI

```bash
copilot plugin install my-team-standards
```

Skills from plugins appear alongside local skills. Disabled plugins disable their skills.

### Community Resources

- **Awesome Copilot:** [github.com/github/awesome-copilot](https://github.com/github/awesome-copilot)
- **Agent Skills Spec:** [agentskills.io](https://agentskills.io)
- **Anthropic Skills:** [github.com/anthropics/skills](https://github.com/anthropics/skills)
- **Community Index:** [github.com/heilcheng/awesome-agent-skills](https://github.com/heilcheng/awesome-agent-skills)

---

## 13. Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════╗
║  SKILL AUTHORING CHECKLIST                                       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  □ Directory: lowercase-with-hyphens, max 64 chars               ║
║  □ File: exactly SKILL.md (case-sensitive)                       ║
║  □ name: matches directory name exactly                          ║
║  □ description: specific what + when + keywords (max 1024 chars) ║
║  □ Body: imperative instructions, not documentation              ║
║  □ References: relative links to separate files (Level 3 loading)║
║  □ Examples: include expected output / behavior                  ║
║  □ Anti-patterns: state what NOT to do                           ║
║  □ Prerequisites: required tools, packages, setup                ║
║  □ allowed-tools: omit unless self-authored + fully trusted      ║
║  □ Test: /skills menu shows it, try invoking via /name           ║
║                                                                  ║
║  PROGRESSIVE LOADING:                                            ║
║    Level 1: name + description (always loaded, minimal cost)     ║
║    Level 2: SKILL.md body (loaded on match)                      ║
║    Level 3: referenced files (loaded on access)                  ║
║                                                                  ║
║  LOCATION:                                                       ║
║    Repo: .github/skills/{name}/SKILL.md                          ║
║    Personal: ~/.copilot/skills/{name}/SKILL.md                   ║
║    Cross-tool: .agents/skills/{name}/SKILL.md                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 14. Key Resources

- **VS Code Skills Docs:** [code.visualstudio.com/docs/copilot/customization/agent-skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- **GitHub Skills Docs:** [docs.github.com/copilot/how-tos/use-copilot-agents/cloud-agent/create-skills](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/create-skills)
- **Copilot CLI Skills:** [github.com/github/copilot-cli-for-beginners/05-skills](https://github.com/github/copilot-cli-for-beginners/blob/main/05-skills/README.md)
- **Customization Handbook:** [copilot-academy.github.io](https://copilot-academy.github.io/workshops/copilot-customization/copilot_customization_handbook)
