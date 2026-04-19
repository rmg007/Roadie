# ⚙️ Extension Manifest & Configuration

# Extension Manifest & Configuration

## package.json Configuration for Phase 1 & Phase 1.5

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
    "vscode": "^1.93.0",
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
    "workspaceContains:.github/.roadie/project-model.db",
    "onStartupFinished"
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
      },
      {
        "command": "roadie.stats",
        "title": "Roadie: Show Stats",
        "description": "Show workflow history statistics from the learning database"
      },
      {
        "command": "roadie.enableWorkflowHistory",
        "title": "Roadie: Enable Workflow History",
        "description": "Start recording every @roadie run to the local learning database"
      },
      {
        "command": "roadie.disableWorkflowHistory",
        "title": "Roadie: Disable Workflow History",
        "description": "Stop recording @roadie runs to the local learning database"
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
            "Use free tier only (cheapest, lower quality)",
            "Free tier → escalate to standard on failure (default)",
            "Start at standard tier (most expensive, highest quality)"
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
    "build": "tsup src/extension.ts --outDir out --format cjs --external vscode",
    "build:watch": "npm run build -- --watch",
    "lint": "eslint src --ext ts",
    "lint:fix": "npm run lint -- --fix",
    "format": "prettier --write src",
    "test": "vitest run",
    "test:watch": "vitest watch",
    "test:coverage": "vitest run --coverage",
    "package": "vsce package",
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
  "bundleDependencies": [
    "better-sqlite3"
  ]
}
```

> **Note:** There is no `"files"` array in `package.json`. Do NOT add one. `vsce` does not support combining `"files"` with `.vscodeignore` — it throws a fatal error. Packaging is controlled exclusively via `.vscodeignore` (see [Build & Packaging](#build--packaging) below).
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

- `"economy"` — Use free tier only. No escalation. Cheapest, lower quality.
- `"balanced"` — Default. Free tier → standard on failure. Good cost/quality balance.
- `"quality"` — Start workflows at standard tier. Most expensive, highest quality.

**Effect:** Changes starting tier assignment. Escalation logic still applies.

**Example:**

```json
{
  "roadie.modelPreference": "quality"
}
```

All workflows start at standard tier instead of free tier.

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

**Trigger:** `Ctrl+Shift+P` (Windows/Linux) / `Cmd+Shift+P` (Mac) → "Roadie: Reset"  

**Action:** Shows a confirmation dialog. On confirm, clears the in-memory project model. On next activation, analysis reruns from scratch.

**Use Case:** Troubleshooting, uninstall prep, or starting fresh.  

### roadie.stats

**Title:** Roadie: Show Stats  

**Trigger:** `Ctrl+Shift+P` → "Roadie: Show Stats"  

**Action:** Reads workflow history stats from `LearningDatabase` and displays a summary notification. Also logs the full breakdown (by workflow type, success rate) to the Roadie Output channel.  

**Fallback:** If SQLite is unavailable (in-memory-only mode), shows a notification explaining that no persistent data has been recorded yet.  

### roadie.enableWorkflowHistory

**Title:** Roadie: Enable Workflow History  

**Trigger:** `Ctrl+Shift+P` → "Roadie: Enable Workflow History"  

**Action (two steps, both happen immediately):**
1. Writes `roadie.workflowHistory = true` to the user's global VS Code settings (persists across workspace and restarts)
2. Hot-updates the live `LearningDatabase` instance via `learningDb.setWorkflowHistory(true)` — takes effect for the current session without a reload

**No settings UI required.** Developer does not need to open VS Code Settings or edit settings.json manually.

**Fallback:** If SQLite is unavailable, saves the setting but shows a warning that the database needs to initialise on next reload.

### roadie.disableWorkflowHistory

**Title:** Roadie: Disable Workflow History  

**Trigger:** `Ctrl+Shift+P` → "Roadie: Disable Workflow History"  

**Action:** Writes `roadie.workflowHistory = false` to global settings and hot-updates the live instance.

**Note:** Existing records in the database are preserved. Only new workflows stop being recorded.

---

## Activation Events

```json
"activationEvents": [
  "onChat:roadie",
  "workspaceContains:.github/.roadie/project-model.db",
  "onStartupFinished"
]
```

**Meaning:**

- `onChat:roadie` — Extension activates when the developer selects `@roadie` from the chat dropdown.
- `workspaceContains:.github/.roadie/project-model.db` — Extension activates automatically if Roadie was previously initialized in this workspace (database exists from a prior session).
- `onStartupFinished` — Extension activates after VS Code finishes starting up, even if neither of the above events occurs. This ensures the startup analysis and `.github/` file generation always run on launch.

**Why `onStartupFinished` was added:** Without it, Roadie would only activate when the user explicitly opened Chat and typed `@roadie`. The `.github/copilot-instructions.md` file would never be generated on a fresh install. `onStartupFinished` fires once after VS Code is fully ready, with no perceptible startup cost.

---

## VS Code Engine Version

```json
"engines": {
  "vscode": "^1.93.0",
  "node": ">=20.0.0"
}
```

**vscode ^1.93.0:** Extension requires VS Code September 2024 or later.

- Includes stable Chat Participant API (graduated from proposed API in 1.93.0)
- Includes stable Language Model API
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
# IMPORTANT: Always use npx @vscode/vsce, NOT a global vsce installation
npx @vscode/vsce package
# Output: roadie-0.5.0.vsix

# Publish to VS Code Marketplace
npm run publish
```

**Do NOT use `--no-dependencies`.** When that flag is set, `vsce` skips the `node_modules` directory entirely — it never walks it, so the `!node_modules/better-sqlite3/...` re-inclusion rules in `.vscodeignore` are never applied. The native binary ends up missing from the `.vsix`. Use plain `vsce package` and let `.vscodeignore` control exactly which `node_modules` files are included.

---

### tsup Build Configuration

The bundler is `tsup`. Key settings in `tsup.config.ts`:

```typescript
export default defineConfig({
  entry: ['src/extension.ts'],
  outDir: 'out',
  format: ['cjs'],
  external: ['vscode', 'better-sqlite3'],     // Do NOT bundle these
  noExternal: ['fast-glob', 'zod'],            // DO inline these into extension.js
});
```

**Why `noExternal: ['fast-glob', 'zod']`?**  
These are pure-JavaScript packages. Inlining them into `out/extension.js` means the `.vsix` doesn't need a `node_modules/fast-glob` or `node_modules/zod` directory at all — saving ~2 MB and eliminating packaging complexity.

**Why `external: ['better-sqlite3']`?**  
`better-sqlite3` contains a native `.node` binary (`better_sqlite3.node`). Native binaries cannot be bundled by any JavaScript bundler — they must exist as separate files on disk. The binary is included in the `.vsix` via `.vscodeignore` rules (see below).

**Why `external: ['vscode']`?**  
The `vscode` module is provided by VS Code at runtime. It is never shipped inside the `.vsix`.

---

### .vscodeignore Strategy

`.vscodeignore` controls which files land in the `.vsix` package. Entries without `!` exclude files; entries with `!` re-include them.

**The canonical, verified `.vscodeignore`:**

```
# ── Source & config (never ship) ─────────────────────────────────────────────
src/**
tsconfig.json
tsup.config.ts
tsup.config.js
vitest.config.ts
vitest.config.js
.eslintrc*
.prettierrc*
*.test.ts
*.spec.ts
test/

# ── Dev tooling files ─────────────────────────────────────────────────────────
.vscode/
.gitignore
AGENTS.md

# ── Generated .github/ files (not needed in the extension package) ────────────
.github/

# ── All node_modules excluded by default ─────────────────────────────────────
node_modules/**

# ── Re-include ONLY the essential better-sqlite3 runtime files ───────────────
# (src/, deps/, binding.gyp stay excluded — we only need the JS wrapper + binary)
!node_modules/better-sqlite3/package.json
!node_modules/better-sqlite3/lib/
!node_modules/better-sqlite3/lib/**
!node_modules/better-sqlite3/build/
!node_modules/better-sqlite3/build/Release/
!node_modules/better-sqlite3/build/Release/better_sqlite3.node

# ── bindings: runtime dependency of better-sqlite3 (locates the .node binary) ─
!node_modules/bindings/
!node_modules/bindings/package.json
!node_modules/bindings/bindings.js

# ── file-uri-to-path: runtime dependency of bindings ─────────────────────────
!node_modules/file-uri-to-path/
!node_modules/file-uri-to-path/package.json
!node_modules/file-uri-to-path/index.js
```

**Result:** The `.vsix` contains ~37 files at ~1.25 MB.

#### Why these three packages?

When `require('better-sqlite3')` runs inside the extension, the execution chain is:

```
better-sqlite3/lib/index.js
  → better-sqlite3/lib/database.js
      → require('bindings')('better_sqlite3.node')   ← locates the binary
          → require('file-uri-to-path')              ← converts file:// URIs on Windows
```

All three packages must be present in the `.vsix`. Missing any one of them produces a `Cannot find module` error at activation time.

#### Why NOT `!node_modules/better-sqlite3/` (directory re-include)?

Using `!node_modules/better-sqlite3/` as a pattern re-includes the entire directory contents including 9 MB of C++ source (`src/**`, `deps/sqlite3.c`) and build artifacts. Use specific file/subdirectory patterns instead — the `.vscodeignore` above is the minimum that makes `require('better-sqlite3')` work.

#### CRITICAL: No `"files"` array in package.json

Do NOT add a `"files"` array to `package.json`. `vsce` throws a fatal error if both exist:
```
ERROR: Both a .vscodeignore file and a 'files' property in package.json were found.
```

---

### better-sqlite3 Version & Electron Compatibility

`better-sqlite3` is a native module — the compiled binary must match VS Code's Electron/V8 version exactly. This is a hard requirement, not optional.

#### Version requirements (verified)

| VS Code version | Electron version | Required better-sqlite3 |
|---|---|---|
| 1.115.0 (April 2026) | 39.8.5 | **v12.x** (v9.x is incompatible) |
| ≤ 1.93.x (Sept 2024) | ~31.x | v9.x may work |

**Why v9 fails on Electron 39:**
- `binding.gyp` in v9.x hardcodes `/std:c++17`, but Electron 39's V8 headers require C++20
- v9.x uses deprecated V8 APIs (`CopyablePersistentTraits`, `AccessorGetterCallback`) that were removed in V8 12.x (bundled in Electron 35+)

**v12.x fixes both:** `binding.gyp` upgraded to `/std:c++20`, and all deprecated V8 API calls were updated.

#### How to find your VS Code's Electron version

```bash
# Read from VS Code's own package.json:
# Windows default install path:
cat "C:\Users\<you>\AppData\Local\Programs\Microsoft VS Code\<hash>\resources\app\package.json" | grep electron
# Look for: "electron": "39.8.5"
```

Or in VS Code: **Help → About** (shows VS Code version; cross-reference with the table above).

#### Rebuilding better-sqlite3 for a target Electron version

Run this from the extension root before packaging:

```bash
cd C:\dev\Roadie\roadie

# 1. Upgrade to a compatible version (v12+ for Electron 35+)
npm install better-sqlite3@latest

# 2. Compile the native binary against VS Code's Electron headers
npm rebuild better-sqlite3 --runtime=electron --target=39.8.5 --dist-url=https://electronjs.org/headers
# Replace 39.8.5 with your VS Code's actual Electron version

# 3. Build and package
npm run build
npx @vscode/vsce package
```

The rebuild downloads the correct Electron headers from `electronjs.org` and compiles `better_sqlite3.node` against them. This binary then works inside VS Code's extension host process.

**CRITICAL:** Do NOT add a `"files"` array to `package.json`. `vsce` throws a fatal error if both `"files"` in `package.json` and `.vscodeignore` are present:  
`ERROR: Both a .vscodeignore file and a 'files' property in package.json were found.`

---

### Command Auto-Update Settings Pattern

When a command needs to change a VS Code setting (e.g. toggling workflow history), use this two-step pattern:

```typescript
// src/shell/commands.ts
export async function updateSetting(key: string, value: unknown): Promise<void> {
  await vscode.workspace
    .getConfiguration('roadie')
    .update(key, value, vscode.ConfigurationTarget.Global);
}
```

```typescript
// extension.ts — inside the command callback
onEnableWorkflowHistory: async () => {
  // Step 1: Persist to VS Code global settings (survives window reload and restarts)
  await updateSetting('workflowHistory', true);

  // Step 2: Hot-update the live service instance (takes effect immediately, no reload needed)
  if (learningDb) {
    learningDb.setWorkflowHistory(true);
    void vscode.window.showInformationMessage('Roadie: Workflow history enabled.');
  }
},
```

**`ConfigurationTarget.Global`** writes to the user's global `settings.json` (not the workspace `.vscode/settings.json`). This means the setting persists across all workspaces and survives VS Code restarts.

**Hot-update without reload:** The service instance (`LearningDatabase`, etc.) exposes a setter method so the command's new value takes effect in the same session without requiring a window reload. The VS Code settings write is the persistence layer; the setter call is the live-session update.

---

## Marketplace Metadata

**Display Name:** Roadie — The Invisible AI Workflow Engine  

**Publisher:** roadie  

**Version:** 0.8.0 (Phase 1 + Phase 1.5)

**Release:** Preview/Pre-release (marked as pre-release on marketplace)  

**Categories:**

- AI
- Code Quality
- Productivity

**Keywords:** copilot, ai, workflow, automation, bug fix, refactoring, code review

---

**Status:** ✅ Phase 1 & Phase 1.5 implemented

**Next:** Go to Phase 1 Project Structure for file/folder layout.