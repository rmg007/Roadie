# ⚙️ Extension Manifest & Configuration

# Extension Manifest & Configuration

## package.json Configuration for Phase 1

---

## Complete package.json

```json
{
  "name": "roadie",
  "displayName": "Roadie — The Invisible AI Workflow Engine",
  "description": "VS Code extension that makes GitHub Copilot smarter. Transforms chat into autonomous workflows: bug fix, feature development, refactoring, code review, documentation, dependency management, onboarding.",
  "version": "0.5.0",
  "publisher": "roadie",
  "engines": {
    "vscode": "^1.84.0",
    "node": ">=20.0.0"
  },
  "categories": [
    "AI",
    "Code Quality",
    "Productivity"
  ],
  "keywords": [
    "copilot",
    "ai",
    "workflow",
    "automation",
    "bug fix",
    "refactoring",
    "code review"
  ],
  "activationEvents": [
    "onChat:roadie",
    "workspaceContains:.github/.roadie/project-model.db"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "chatParticipants": [
      {
        "id": "roadie",
        "name": "Roadie",
        "description": "The invisible AI workflow engine. Chat naturally—Roadie handles the rest.",
        "isSticky": true
      }
    ],
    "commands": [
      {
        "command": "roadie.init",
        "title": "Roadie: Initialize",
        "description": "Manually initialize Roadie in the workspace"
      },
      {
        "command": "roadie.rescan",
        "title": "Roadie: Rescan Project",
        "description": "Force a full project rescan"
      },
      {
        "command": "roadie.reset",
        "title": "Roadie: Reset",
        "description": "Delete local database and reset Roadie"
      }
    ],
    "configuration": {
      "title": "Roadie",
      "properties": {
        "roadie.telemetry": {
          "type": "boolean",
          "default": false,
          "description": "Enable anonymous, aggregate telemetry (workflow types, model tiers, success rates). Never sends code, file names, or project details."
        },
        "roadie.editTracking": {
          "type": "boolean",
          "default": false,
          "description": "Track edits to Roadie-generated files for learning preferences. Phase 1.5 only."
        },
        "roadie.workflowHistory": {
          "type": "boolean",
          "default": false,
          "description": "Persist workflow outcomes (success/failure, escalation patterns). Phase 1.5 only."
        },
        "roadie.modelPreference": {
          "type": "string",
          "enum": [
            "economy",
            "balanced",
            "quality"
          ],
          "default": "balanced",
          "enumDescriptions": [
            "Use Tier 0 only (cheapest, lower quality)",
            "Tier 0 → escalate to Tier 1 on failure (default)",
            "Start at Tier 1 (most expensive, highest quality)"
          ],
          "description": "Model tier preference for workflows. Does not affect escalation logic."
        },
        "roadie.autoCommit": {
          "type": "boolean",
          "default": false,
          "description": "Automatically stage and commit Roadie-generated .github/ files. Phase 1.5 only."
        },
        "roadie.testCommand": {
          "type": "string",
          "default": "",
          "description": "Custom test command override. If empty, Roadie auto-detects from package.json scripts (e.g., 'npm test', 'pnpm test')."
        },
        "roadie.testTimeout": {
          "type": "number",
          "default": 300,
          "minimum": 10,
          "maximum": 3600,
          "description": "Maximum seconds to wait for test suite execution before timeout."
        }
      }
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run build",
    "build": "tsup src/extension.ts --outDir out --format cjs --external:vscode",
    "build:watch": "npm run build -- --watch",
    "lint": "eslint src --ext ts",
    "lint:fix": "npm run lint -- --fix",
    "format": "prettier --write src",
    "test": "vitest run",
    "test:watch": "vitest watch",
    "test:coverage": "vitest run --coverage",
    "package": "vsce package --no-dependencies",
    "publish": "vsce publish",
    "prepublish:test": "npm run test && npm run lint && npm run build"
  },
  "dependencies": {
    "better-sqlite3": "^9.4.3",
    "fast-glob": "^3.3.0",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.0",
    "@types/node": "^20.0.0",
    "@types/vscode": "^1.84.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "@vscode/test-cli": "^0.0.4",
    "@vscode/test-electron": "^2.3.0",
    "eslint": "^8.50.0",
    "prettier": "^3.0.0",
    "tsup": "^8.0.0",
    "typescript": "^5.2.0",
    "vitest": "^0.34.0"
  },
  "files": [
    "out",
    "package.json",
    "README.md",
    "LICENSE"
  ]
}
```

---

## Configuration Properties Explained

### roadie.telemetry (boolean, default: false)

**What gets sent:**

- Workflow types triggered (e.g., "bug_fix ran 3 times this week")
- Model tiers used (percentage distribution)
- Success/failure rates per workflow
- Extension version, VS Code version

**What NEVER gets sent:**

- Code content
- File names or paths
- Project identifiers
- Personal information
- Chat history

**Privacy:** Data is anonymous and aggregate. Cannot identify developer or project.

---

### roadie.editTracking (boolean, default: false)

**Phase 1.5 feature** (stub in Phase 1, enabled in 1.5)  

**Effect:** Stores diffs of developer edits to generated files for preference learning.  

**Privacy:** Data stored locally only; never transmitted.  

---

### roadie.workflowHistory (boolean, default: false)

**Phase 1.5 feature** (stub in Phase 1)  

**Effect:** Persists workflow outcomes for pattern learning and optimization.  

**Privacy:** Data stored locally only; never transmitted.  

---

### roadie.modelPreference (string, default: "balanced")

**Values:**

- `"economy"` — Use Tier 0 (free) only. No escalation. Cheapest, lower quality.
- `"balanced"` — Default. Tier 0 → Tier 1 on failure. Good cost/quality balance.
- `"quality"` — Start workflows at Tier 1. Most expensive, highest quality.

**Effect:** Changes starting tier assignment. Escalation logic still applies.

**Example:**

```json
{
  "roadie.modelPreference": "quality"
}
```

All workflows start at Tier 1 instead of Tier 0.

---

### roadie.autoCommit (boolean, default: false)

**Phase 1.5 feature** (stub in Phase 1)  

**Effect:** Stages and commits generated `.github/` files automatically.  

**Commit Message:** `chore(roadie): update AI configuration [skip ci]`  

**Details:** Never amends existing commits; always creates new commits.  

---

### roadie.testCommand (string, default: "")

**Effect:** Command Roadie runs for the "run tests" step in bug-fix, feature, refactor, and review workflows.

**Detection Algorithm** (applied when `roadie.testCommand` is empty):

```tsx
// src/shell/test-command-detector.ts

export type TestCommandResult =
  | { found: true;  command: string; source: string }
  | { found: false; reason: 'no_manifest' | 'no_test_script' | 'language_unsupported' };

/**
 * Canonical decision tree — execute top-to-bottom, return the first match.
 * If every branch fails, return `{ found: false, ... }` and the workflow engine
 * surfaces a notification asking the user to set `roadie.testCommand` manually.
 */
export async function detectTestCommand(workspaceRoot: string): Promise<TestCommandResult> {
  // 0. User override always wins (handled by caller — detector is only invoked when the setting is empty).

  // 1. JavaScript / TypeScript — package.json scripts
  const pkg = await readJsonIfExists(join(workspaceRoot, 'package.json'));
  if (pkg && pkg.scripts && typeof pkg.scripts === 'object') {
    // Priority order is fixed and must match test/test-command-detector.test.ts
    const SCRIPT_PRIORITY = ['test', 'test:ci', 'test:unit', 'test:all'] as const;
    for (const name of SCRIPT_PRIORITY) {
      if (typeof pkg.scripts[name] === 'string') {
        const pm = await detectPackageManager(workspaceRoot);
        return { found: true, command: `${pm} run ${name}`, source: `package.json#scripts.${name}` };
      }
    }
    // package.json exists but has no test script → fall through to framework detection below
  }

  // 2. Python — pyproject.toml (PEP 518)
  if (await fileExists(join(workspaceRoot, 'pyproject.toml'))) {
    return { found: true, command: 'pytest', source: 'pyproject.toml' };
  }
  // Python — legacy fallback
  if (await fileExists(join(workspaceRoot, 'setup.py')) || await fileExists(join(workspaceRoot, 'tox.ini'))) {
    return { found: true, command: 'pytest', source: 'setup.py|tox.ini' };
  }

  // 3. Rust — Cargo.toml
  if (await fileExists(join(workspaceRoot, 'Cargo.toml'))) {
    return { found: true, command: 'cargo test', source: 'Cargo.toml' };
  }

  // 4. Go — go.mod
  if (await fileExists(join(workspaceRoot, 'go.mod'))) {
    return { found: true, command: 'go test ./...', source: 'go.mod' };
  }

  // 5. Ruby — Gemfile + rake or rspec
  if (await fileExists(join(workspaceRoot, 'Gemfile'))) {
    if (await fileExists(join(workspaceRoot, '.rspec'))) {
      return { found: true, command: 'bundle exec rspec', source: 'Gemfile+.rspec' };
    }
    if (await fileExists(join(workspaceRoot, 'Rakefile'))) {
      return { found: true, command: 'bundle exec rake test', source: 'Gemfile+Rakefile' };
    }
  }

  // 6. Java / Kotlin — Gradle or Maven
  if (await fileExists(join(workspaceRoot, 'build.gradle')) || await fileExists(join(workspaceRoot, 'build.gradle.kts'))) {
    return { found: true, command: './gradlew test', source: 'build.gradle' };
  }
  if (await fileExists(join(workspaceRoot, 'pom.xml'))) {
    return { found: true, command: 'mvn test', source: 'pom.xml' };
  }

  // 7. Nothing matched
  if (!pkg) return { found: false, reason: 'no_manifest' };
  return { found: false, reason: 'no_test_script' };
}

/**
 * Detect JavaScript package manager. Priority: pnpm → yarn → bun → npm.
 * The first lockfile wins.
 */
async function detectPackageManager(workspaceRoot: string): Promise<'pnpm' | 'yarn' | 'bun' | 'npm'> {
  if (await fileExists(join(workspaceRoot, 'pnpm-lock.yaml'))) return 'pnpm';
  if (await fileExists(join(workspaceRoot, 'yarn.lock')))      return 'yarn';
  if (await fileExists(join(workspaceRoot, 'bun.lockb')))      return 'bun';
  return 'npm';
}
```

**Failure mode:** If `detectTestCommand` returns `{ found: false }`, the workflow engine calls:

```ts
const action = await vscode.window.showWarningMessage(
  `Roadie couldn't auto-detect a test command for this workspace. Set "roadie.testCommand" in settings to continue.`,
  'Open Settings',
  'Skip Test Step',
);
```

- **"Open Settings"** → `vscode.commands.executeCommand('workbench.action.openSettings', 'roadie.testCommand')`.
- **"Skip Test Step"** → workflow marks Step 4 as `skipped` and proceeds to Step 5 with a visible warning in the chat stream. The workflow's final result reports `testsRun: false`.
- **Dismissing the notification** is equivalent to "Skip Test Step".

**Determinism requirement:** The decision tree is total-ordered. Two invocations on the same workspace MUST return the same result. Tests assert this by snapshotting `(workspaceRoot, expectedCommand, expectedSource)` tuples in `test/fixtures/test-command-detection/`.

---

### roadie.testTimeout (number, default: 300 seconds)

**Range:** 10-3600 seconds  

**Effect:** Maximum time to wait for test suite execution (workflow step 4 in bug-fix, feature, refactor).  

**Example:**

```json
{
  "roadie.testTimeout": 60
}
```

Test suite times out after 60 seconds.

---

## Chat Participant Registration

```json
"chatParticipants": [
  {
    "id": "roadie",
    "name": "Roadie",
    "description": "The invisible AI workflow engine. Chat naturally—Roadie handles the rest.",
    "isSticky": true
  }
]
```

**isSticky: true** means Roadie stays selected in the chat dropdown after the first selection (user doesn't have to re-select for every message).

---

## Commands

### roadie.init

**Title:** Roadie: Initialize  

**Trigger:** `Cmd+Shift+P` → "Roadie: Initialize"  

**Action:** Manually initialize Roadie in the current workspace.  

**Use Case:** If extension somehow didn't auto-initialize, or for testing.  

### roadie.rescan

**Title:** Roadie: Rescan Project  

**Trigger:** `Cmd+Shift+P` → "Roadie: Rescan Project"  

**Action:** Force a full project model rebuild from scratch.  

**Use Case:** If project structure changed significantly, force cache invalidation.  

### roadie.reset

**Title:** Roadie: Reset  

**Trigger:** `Cmd+Shift+P` → "Roadie: Reset"  

**Action:** Delete `.github/.roadie/project-model.db` and completely reset extension state.  

**Use Case:** Troubleshooting, uninstall prep, or starting fresh.  

---

## Activation Events

```json
"activationEvents": [
  "onChat:roadie",
  "workspaceContains:.github/.roadie/project-model.db"
]
```

**Meaning:**

- `onChat:roadie` — Extension activates when the developer selects `@roadie` from the chat dropdown.
- `workspaceContains:.github/.roadie/project-model.db` — Extension activates automatically if Roadie was previously initialized in this workspace (database exists from a prior session).

**Lazy Activation:** Extension doesn't load until one of these conditions is met. Minimal startup impact on VS Code.

---

## VS Code Engine Version

```json
"engines": {
  "vscode": "^1.84.0",
  "node": ">=20.0.0"
}
```

**vscode ^1.84.0:** Extension requires VS Code November 2023 or later.

- Includes stable Chat Participant API
- Includes Language Model API
- Includes FileSystemWatcher reliability improvements

**node >=20.0.0:** Extension requires Node.js 20+.

- TypeScript support
- Better-sqlite3 pre-built binaries available

---

## Build & Packaging

### Local Development

```bash
# Install dependencies
npm install

# Build for development
npm run build

# Run tests
npm run test

# Lint
npm run lint

# Watch mode (auto-rebuild on changes)
npm run build:watch

# Launch Extension Development Host (F5 in VS Code)
```

### Packaging for Marketplace

```bash
# Full pre-publish checks
npm run prepublish:test

# Package into .vsix file
npm run package
# Output: roadie-0.5.0.vsix

# Publish to VS Code Marketplace
npm run publish
```

---

## Marketplace Metadata

**Display Name:** Roadie — The Invisible AI Workflow Engine  

**Publisher:** roadie  

**Version:** 0.5.0 (Phase 1)

**Release:** Preview/Pre-release (marked as pre-release on marketplace)  

**Categories:**

- AI
- Code Quality
- Productivity

**Keywords:** copilot, ai, workflow, automation, bug fix, refactoring, code review

---

**Next:** Go to Phase 1 Project Structure for file/folder layout.