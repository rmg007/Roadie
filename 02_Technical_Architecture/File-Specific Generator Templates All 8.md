# 📦 File-Specific Generator Templates (All 8)

## Template Structure, Section Definitions, and Content Generation for All 8 File Types

---

## Common Pattern

Every generator sub-module implements the same `FileTypeGenerator` interface:

```tsx
interface FileTypeGenerator {
  fileType: GeneratedFileType;
  triggers: string[];  // Which model change categories trigger regeneration
  generate(model: ProjectModel): Promise<GeneratedContent>;
}

interface GeneratedContent {
  filePath: string;
  sections: GeneratedSection[];
}

interface GeneratedSection {
  id: string;          // Unique section ID (used in markers)
  content: string;     // Generated Markdown/YAML/etc.
  priority: 'required' | 'recommended' | 'optional';
}
```

Every generator:

- Reads from `ProjectModel` (never from the file system directly)
- Returns `GeneratedSection[]` (the Section Manager handles markers, hashing, and merge)
- Uses TypeScript template strings (no template engine)
- Completes in <250ms
- Is pure and deterministic (same model input → same output)

---

## 1. Copilot Instructions Generator

**File:** `src/generator/templates/copilot-instructions.ts`

**Output:** `.github/copilot-instructions.md`

**Triggers:** `techStack`, `patterns`, `commands`, `config`

**Estimated lines:** ~150

### Sections

| Section ID | Priority | Content |
| --- | --- | --- |
| `project-overview` | required | Project name, description (from package.json), primary language |
| `tech-stack` | required | Languages, frameworks, runtimes with versions |
| `coding-standards` | recommended | Detected patterns: export style, test conventions, error handling, import ordering |
| `project-structure` | recommended | Top-level directory layout with roles (src=source, test=tests, etc.) |
| `commands` | required | Build, test, dev, lint commands from package.json scripts |
| `conventions` | optional | Commit convention, naming patterns (if detected with >0.7 confidence) |

### Content Generation

```tsx
function generateTechStackSection(model: ProjectModel): string {
  const stack = model.getTechStack();
  const grouped = groupBy(stack, 'category');
  
  let content = '## Tech Stack\n\n';
  for (const [category, entries] of Object.entries(grouped)) {
    content += `### ${capitalize(category)}\n`;
    for (const entry of entries) {
      content += `- **${entry.name}**`;
      if (entry.version) content += ` ${entry.version}`;
      content += '\n';
    }
    content += '\n';
  }
  return content;
}

function generatePatternsSection(model: ProjectModel): string {
  const patterns = model.getPatterns().filter(p => p.confidence >= 0.7);
  if (patterns.length === 0) return '';
  
  let content = '## Coding Standards\n\n';
  for (const pattern of patterns) {
    content += `- **${pattern.category}:** ${pattern.description}\n`;
  }
  return content;
}

function generateCommandsSection(model: ProjectModel): string {
  const commands = model.getCommands();
  if (commands.length === 0) return '';
  
  let content = '## Commands\n\n';
  content += '```bash\n';
  for (const cmd of commands) {
    content += `# ${cmd.type}\n${cmd.command}\n\n`;
  }
  content += '```\n';
  return content;
}
```

### Rules

- Keep under 100 lines of output (ideally 50-70)
- Only include patterns detected with confidence ≥0.7
- Never include sensitive information (env vars, secrets)
- No frontmatter — pure Markdown
- Use concrete examples from the actual codebase where possible

---

## 2. Path Instructions Generator

**File:** `src/generator/templates/path-instructions.ts`

**Output:** `.github/instructions/{path}.instructions.md` (one per major directory)

**Triggers:** `structure`, `patterns`

**Estimated lines:** ~120

### How It Works

1. Scan `directoryStructure` for top-level source directories
2. For each directory with a clear role (components, pages, api, utils, etc.), generate a path-specific instruction file
3. Each file gets YAML frontmatter with `applyTo` glob

### Section per File

| Section ID | Priority | Content |
| --- | --- | --- |
| `path-context` | required | What this directory contains, naming conventions, patterns specific to this path |

### Content Generation

```tsx
function generate(model: ProjectModel): Promise<GeneratedContent[]> {
  const structure = model.getDirectoryStructure();
  const results: GeneratedContent[] = [];
  
  for (const dir of getTopLevelSourceDirs(structure)) {
    const role = inferDirectoryRole(dir); // 'components', 'pages', 'api', 'utils', etc.
    if (!role) continue;
    
    const content = generatePathInstructions(dir, role, model);
    results.push({
      filePath: `.github/instructions/${dir.path.replace(/\//g, '-')}.instructions.md`,
      sections: [{
        id: 'path-context',
        content: `---\napplyTo: "${dir.path}/**"\n---\n\n${content}`,
        priority: 'required'
      }]
    });
  }
  return results;
}
```

### Directory Role Templates

| Role | Generated Content Focus |
| --- | --- |
| `components` | Component patterns, prop conventions, styling approach |
| `pages`/`routes` | Routing patterns, data fetching, layout conventions |
| `api`/`routes` | API patterns, request/response conventions, middleware |
| `utils`/`lib` | Utility function patterns, pure function conventions |
| `tests` | Test framework, naming conventions, fixture patterns |
| `models`/`entities` | Data model patterns, ORM conventions, validation |
| `services` | Service layer patterns, dependency injection |

---

## 3. Agent Definition Generator

**File:** `src/generator/templates/agent-definitions.ts`

**Output:** `.github/agents/{role}.agent.md` (one per workflow type)

**Triggers:** `techStack`, `patterns`

**Estimated lines:** ~180

### Generated Agents

| Agent File | Role | Tools | Model Tier |
| --- | --- | --- | --- |
| `debugger.agent.md` | Bug diagnosis and fixing | search, read, edit, terminal | standard |
| `reviewer.agent.md` | Multi-perspective code review | search, read, terminal | premium |
| `planner.agent.md` | Feature planning (read-only) | search, read, web | premium |
| `implementer.agent.md` | Code implementation | edit, create, delete, terminal, search, read | standard |
| `documenter.agent.md` | Documentation generation | search, read, edit, create | standard |

### Content Generation

Each agent file includes:

- YAML frontmatter (name, description, tools, model)
- Role-specific system prompt
- Project-aware context (injected from project model: tech stack, conventions)
- Tool scoping (least privilege principle)

```tsx
function generateDebuggerAgent(model: ProjectModel): string {
  const stack = model.getTechStack();
  const testCmd = model.getCommands().find(c => c.type === 'test');
  
  return `---
name: Debugger
description: Diagnoses and fixes bugs. Runs tests to verify fixes.
tools: ['search/codebase', 'read_file', 'edit_file', 'terminal']
model: 'Claude Sonnet 4.6'
---

# Bug Fix Agent

You diagnose and fix bugs in a ${stack[0]?.name || 'TypeScript'} project.

## Workflow
1. Locate the error source using search and file reading
2. Diagnose the root cause
3. Apply the minimal fix
4. Run tests: \`${testCmd?.command || 'npm test'}\`
5. If tests fail, iterate on the fix
6. Search for similar patterns elsewhere in the codebase

## Rules
- Always run tests after fixing
- Prefer minimal, targeted fixes over rewrites
- Check for the same bug pattern in related files
`;
}
```

---

## 4. Skills Generator

**File:** `src/generator/templates/skill-definitions.ts`

**Output:** `.github/skills/{name}/SKILL.md`

**Triggers:** `techStack`, `commands`, `patterns`

**Estimated lines:** ~120

### Generated Skills

| Skill | Trigger | Content |
| --- | --- | --- |
| `test-and-verify` | Test framework detected | Step-by-step test writing procedure for this project's framework |
| `lint-and-format` | Linter/formatter detected | How to run lint, fix issues, format code |
| `dependency-update` | Package manager detected | How to safely update a dependency |

### Content Generation

```tsx
function generateTestSkill(model: ProjectModel): GeneratedContent | null {
  const testFramework = model.getTechStack().find(s => s.category === 'test_tool');
  if (!testFramework) return null;
  
  const testCmd = model.getCommands().find(c => c.type === 'test');
  
  return {
    filePath: `.github/skills/test-and-verify/SKILL.md`,
    sections: [{
      id: 'test-skill',
      content: `---\nname: test-and-verify\ndescription: >-\n  Write and run tests using ${testFramework.name}. Use when asked to\n  add tests, verify fixes, or check test coverage.\n---\n\n# Test & Verify\n\n1. Write test files in the project's test directory\n2. Follow naming convention: \`{module}.test.${getExtension(model)}\`\n3. Run: \`${testCmd?.command || 'npm test'}\`\n4. Verify all tests pass before completing\n`,
      priority: 'recommended'
    }]
  };
}
```

---

## 5. Hooks Generator

**File:** `src/generator/templates/hooks.ts`

**Output:** `.github/hooks/*.json`

**Triggers:** `techStack` (linter/formatter detection)

**Estimated lines:** ~80

### Generated Hooks

| Hook | Trigger | Action |
| --- | --- | --- |
| `format-on-save.json` | Formatter detected (prettier, eslint --fix) | Auto-format after file edit |
| `lint-on-edit.json` | Linter detected | Run linter after code changes |

### Content Generation

```tsx
function generateFormatHook(model: ProjectModel): GeneratedContent | null {
  const formatter = model.getTechStack().find(
    s => s.name === 'prettier' || s.name === 'eslint'
  );
  if (!formatter) return null;
  
  const formatCmd = formatter.name === 'prettier'
    ? 'npx prettier --write {file}'
    : 'npx eslint --fix {file}';
  
  return {
    filePath: '.github/hooks/format-on-save.json',
    sections: [{
      id: 'format-hook',
      content: JSON.stringify({
        event: 'PostToolUse',
        tools: ['edit_file', 'create_file'],
        steps: [{
          type: 'command',
          command: formatCmd
        }]
      }, null, 2),
      priority: 'recommended'
    }]
  };
}
```

---

## 6. Workflows Generator

**File:** `src/generator/templates/workflows.ts`

**Output:** `.github/workflows/pr-test.yml`

**Triggers:** `techStack`, `commands`

**Estimated lines:** ~100

### Generated Workflows

| Workflow | Condition | Content |
| --- | --- | --- |
| `pr-test.yml` | Test command exists AND no existing CI workflow | PR test runner |

### Content Generation

```tsx
function generatePRTestWorkflow(model: ProjectModel): GeneratedContent | null {
  const testCmd = model.getCommands().find(c => c.type === 'test');
  const buildCmd = model.getCommands().find(c => c.type === 'build');
  const lintCmd = model.getCommands().find(c => c.type === 'lint');
  if (!testCmd) return null;
  
  const nodeVersion = model.getTechStack().find(s => s.name === 'Node.js')?.version || '20';
  const pkgManager = model.getTechStack().find(s => s.category === 'package_manager')?.name || 'npm';
  const installCmd = pkgManager === 'pnpm' ? 'pnpm install' : pkgManager === 'yarn' ? 'yarn install' : 'npm ci';
  
  const steps = [
    `      - uses: actions/checkout@v4`,
    `      - uses: actions/setup-node@v4\n        with:\n          node-version: '${nodeVersion}'`,
    `      - run: ${installCmd}`,
  ];
  if (lintCmd) steps.push(`      - run: ${lintCmd.command}`);
  if (buildCmd) steps.push(`      - run: ${buildCmd.command}`);
  steps.push(`      - run: ${testCmd.command}`);
  
  const yaml = `name: PR Tests\non:\n  pull_request:\n    branches: [main, master]\n\njobs:\n  test:\n    runs-on: ubuntu-latest\n    steps:\n${steps.join('\n')}`;
  
  return {
    filePath: '.github/workflows/pr-test.yml',
    sections: [{ id: 'pr-test', content: yaml, priority: 'recommended' }]
  };
}
```

**Important:** Only generate if no existing CI workflow is detected. Check for `.github/workflows/*.yml` before writing.

---

## 7. Templates Generator

**File:** `src/generator/templates/issue-pr-templates.ts`

**Output:** `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/bug_report.md`, `.github/ISSUE_TEMPLATE/feature_request.md`

**Triggers:** `techStack` (generated once, rarely changes)

**Estimated lines:** ~100

### Content

These are mostly static templates with minimal project-specific content.

**PR Template sections:**

| Section ID | Content |
| --- | --- |
| `pr-template` | Description, related issue, checklist (tests, docs, migration) |

**Issue Templates:**

| Section ID | Content |
| --- | --- |
| `bug-report` | Reproduction steps, expected vs actual, environment |
| `feature-request` | Problem statement, proposed solution, alternatives |

---

## 8. [AGENTS.md](http://AGENTS.md) Generator

**File:** `src/generator/templates/agents-md.ts`

**Output:** `AGENTS.md` (project root)

**Triggers:** `techStack`, `structure`, `patterns`

**Estimated lines:** ~100

### Sections

| Section ID | Priority | Content |
| --- | --- | --- |
| `project-overview` | required | What this project is, primary tech stack |
| `architecture` | recommended | High-level architecture and directory roles |
| `conventions` | recommended | Key coding conventions for all AI agents |
| `mcp-tools` | optional | Available MCP tools (added by Phase 2 generator) |

### Content Generation

```tsx
function generateAgentsMd(model: ProjectModel): GeneratedContent {
  const stack = model.getTechStack();
  const structure = model.getDirectoryStructure();
  const patterns = model.getPatterns().filter(p => p.confidence >= 0.7);
  
  const sections: GeneratedSection[] = [
    {
      id: 'project-overview',
      content: `# AGENTS.md\n\nThis is a ${getPrimaryLanguage(stack)} project using ${getPrimaryFramework(stack)}.\n`,
      priority: 'required'
    },
    {
      id: 'architecture',
      content: generateArchitectureSection(structure),
      priority: 'recommended'
    },
    {
      id: 'conventions',
      content: generateConventionsSection(patterns),
      priority: 'recommended'
    }
  ];
  
  return { filePath: 'AGENTS.md', sections };
}
```

---

## Build Prompt for AI Agent

```
Build all 8 file-specific generator sub-modules according to this spec.

Each generator implements the FileTypeGenerator interface.
Each returns GeneratedSection[] that the Section Manager handles.
All use TypeScript template strings (no template engine).
All must complete in <250ms.

Files to create:
- src/generator/templates/copilot-instructions.ts (~150 lines)
- src/generator/templates/path-instructions.ts (~120 lines)
- src/generator/templates/agent-definitions.ts (~180 lines)
- src/generator/templates/skill-definitions.ts (~120 lines)
- src/generator/templates/hooks.ts (~80 lines)
- src/generator/templates/workflows.ts (~100 lines)
- src/generator/templates/issue-pr-templates.ts (~100 lines)
- src/generator/templates/agents-md.ts (~100 lines)
- test/generator/templates/*.test.ts (10+ tests per generator)

Verification:
- Each generator produces correct content for the Node.js/TypeScript test fixture project
- Section IDs are unique across ALL generators (no collisions)
- All generators registered in FileGeneratorManager
- npm run test passes
- Total generation time for all 8 generators < 2s
```

---

**Dependencies:** All generators depend on File Generator Manager (M19) and Section Manager (M22). Build the managers first, then generators.