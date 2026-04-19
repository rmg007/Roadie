# Installation & Quick Start Guide

Two audiences are covered here:

- **Developers** building or iterating on the extension (TypeScript source → Extension Development Host).
- **End users** installing a packaged `.vsix` file into their own VS Code.

---

## Part 1: Developer Setup

### Prerequisites

| Requirement | Minimum version |
|---|---|
| Node.js | 20+ |
| VS Code | 1.93+ |
| GitHub Copilot extension | Installed and signed in |

> `better-sqlite3` requires native compilation. Pre-built binaries are available for Node.js 20, so a plain `npm install` is sufficient on a supported platform.

### Install dependencies and build

```bash
cd C:\dev\Roadie\roadie   # or your local clone
npm install
npm run build             # compiles TypeScript via tsup → out/extension.js
```

### Launch in the Extension Development Host

1. Open `C:\dev\Roadie\roadie` in VS Code.
2. Press **F5** (or **Run → Start Debugging**).
3. A second VS Code window opens — this is the Extension Development Host.
4. In that window, open any project folder.
5. Confirm the status bar at the bottom shows **Roadie active**.
6. Open VS Code Chat (**Ctrl+Alt+I** or **View → Chat**) and type `@roadie hello`.

### Watch mode

```bash
npm run build:watch   # rebuilds on every save
```

After a rebuild, press **Ctrl+Shift+F5** to reload the Extension Development Host without restarting the full debug session.

---

## Part 2: End-User Installation (from .vsix)

### Recommended: plug-and-play installer

From the implementation repo (`C:\dev\Roadie\roadie`):

```bash
npm install
npm run build
npm run install:all
```

`npm run install:all` runs `scripts/install.js`, which:

1. Verifies prerequisites (Node 20+, npm, VS Code CLI).
2. Builds if `out/extension.js` or `out/bin/roadie-mcp.js` is missing.
3. Smoke-tests the MCP server over stdio (real JSON-RPC `initialize`, must respond within 3 s).
4. Installs the packaged `.vsix` via `code --install-extension`.
5. Registers the Roadie MCP server in `~/.claude.json` (Claude Code) and the Claude Desktop config so MCP-capable clients can use Roadie standalone.

Useful flags:

- `--skip-extension` — register the MCP server only (Claude Code-only users).
- `--skip-mcp` — install the VS Code extension only.
- `--uninstall` — reverse both: `code --uninstall-extension roadie.roadie` and remove the MCP entries.
- `--log-level LEVEL` — sets `ROADIE_LOG_LEVEL` written into the MCP entries.

Verify with `npm run doctor` — it re-runs the smoke test, checks `code --list-extensions` for `roadie.roadie`, and confirms the MCP entry is in `~/.claude.json`. Exit code 0 = green.

The installer is idempotent and writes JSON configs atomically with timestamped backups. Safe to re-run.

### Manual fallback

If the installer cannot run (e.g. you only have the `.vsix`, no Node toolchain handy), follow the manual steps below.

### Step 1: Package the extension

Before packaging, ensure `better-sqlite3` has been compiled for VS Code's Electron version (see [Electron ABI compatibility](#electron-abi-compatibility) below). Then run:

```bash
cd C:\dev\Roadie\roadie
npx @vscode/vsce package
```

This produces a file such as `roadie-1.0.0.vsix` in the same directory.

Important notes:

- Use `npx @vscode/vsce`, not `vsce` directly — `vsce` may not be globally installed.
- Do **NOT** use `--no-dependencies`. With that flag, `vsce` skips the `node_modules` directory entirely, so the `better-sqlite3` binary never makes it into the `.vsix`. Use plain `vsce package` and let `.vscodeignore` control exactly which files are included.

### Step 2: Install the .vsix

**Option A — Command Palette**

1. Press **Ctrl+Shift+P** and run **Extensions: Install from VSIX...**
2. Select the `.vsix` file.
3. Reload VS Code when prompted.

**Option B — Command line**

```bash
code --install-extension roadie-1.0.0.vsix
```

### Step 3: Verify installation

1. Open a project folder in VS Code (a folder, not a bare file).
2. The status bar at the bottom of the window should show **Roadie active**.
3. Open the Output channel: **View → Output**, then select **Roadie** from the dropdown.

Expected log output on a successful activation:

```
2026-04-17 09:23:45.123 [INFO] Roadie v1.0.0 activating…
2026-04-17 09:23:45.156 [INFO] Workspace root: C:\my-project
2026-04-17 09:23:45.201 [INFO] Starting startup project analysis…
2026-04-17 09:23:46.304 [INFO] Startup analysis complete — 12 tech entries, 4 commands
2026-04-17 09:23:46.891 [INFO] Files written: .github/copilot-instructions.md, AGENTS.md, CLAUDE.md, .cursor/rules/project.mdc
2026-04-17 09:23:46.895 [INFO] Roadie activated ✓
```

### Step 4: Use @roadie in chat

1. Open VS Code Chat: **Ctrl+Alt+I** (or **View → Chat**).
2. Type `@roadie` — Roadie should appear in the participant list.
3. Send a message, for example:
   - `@roadie fix the login bug`
   - `@roadie add dark mode support`
4. Roadie classifies your intent and runs an autonomous workflow.

---

## Generated Files

On first activation with a workspace open, Roadie creates the following files by analysing your project (package.json, directory structure, etc.):

| File | Purpose |
|---|---|
| `.github/copilot-instructions.md` | Project context injected into GitHub Copilot |
| `.github/AGENTS.md` | Agent role definitions |
| `.github/.roadie/.gitignore` | Excludes the SQLite database from git |

All three files are safe to commit to version control.

---

## Automatic Re-analysis

Roadie watches your workspace for changes to dependency and config files. When a relevant file changes, Roadie automatically re-runs the full project analysis and regenerates `.github/` files — no manual rescan needed.

### What triggers automatic re-analysis

| File changed | Why it matters |
|---|---|
| `package.json` | Scripts, dependencies, project name |
| `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb` | Package manager detection |
| `tsconfig.json`, `tsconfig.*.json` | TypeScript configuration |
| `jest.config.*`, `vitest.config.*` | Test runner detection |
| `vite.config.*`, `webpack.config.*`, `rollup.config.*` | Build tool detection |
| `.babelrc`, `.eslintrc*`, `.prettierrc*` | Linter/formatter detection |

Changes to `node_modules/`, `.git/`, `dist/`, `build/`, and other generated directories are ignored.

### Package manager auto-correction

A common scenario: you switch from npm to pnpm mid-project.

1. Run `pnpm import` — `pnpm-lock.yaml` appears in the workspace
2. Roadie detects it within 500 ms (debounce window)
3. Re-analysis runs: `detectPackageManager()` now returns `'pnpm'`
4. All stored commands update from `npm run …` to `pnpm run …`
5. `.github/copilot-instructions.md` is rewritten automatically

The Output channel confirms the update:

```
[INFO]  File watcher: pnpm-lock.yaml changed — re-analysing…
[INFO]  File watcher: re-analysis complete — 8 commands
[INFO]  File watcher: .github/ updated — .github/copilot-instructions.md
```

### Large changesets (e.g. `git checkout`)

If more than 1000 files change at once (branch switch, `npm install` on a fresh clone), Roadie detects the batch overflow and runs a full rescan automatically:

```
[INFO]  File watcher: batch overflow — running full rescan…
```

### Manual rescan

If you ever need to force a rescan (e.g. the watcher missed an event):

**Ctrl+Shift+P** → **Roadie: Rescan Project**

---

## Workflow History (opt-in)

Workflow runs are not recorded by default. To enable recording:

1. **Ctrl+Shift+P** → **Roadie: Enable Workflow History**

This updates your VS Code global settings automatically — no manual `settings.json` editing required. Workflows are recorded to `.github/.roadie/project-model.db`.

Additional commands:

- **Roadie: Show Stats** — view workflow statistics.
- **Roadie: Disable Workflow History** — stop recording and clear the setting.

---

## Troubleshooting

### "Roadie active" not showing in status bar

- Confirm you opened a **folder**, not just a single file. VS Code requires a workspace for Roadie to activate.
- Reload the window: **Ctrl+Shift+P** → **Developer: Reload Window**.

### No files generated after activation

- Check the Output channel (**View → Output → Roadie**) for error messages.
- If the log says `SQLite unavailable` — this is non-fatal. Roadie runs in memory-only mode and still generates files.
- If analysis reports `0 tech entries` — confirm your project contains a `package.json` or another recognised manifest.

### @roadie not appearing in Chat

- Confirm the GitHub Copilot extension is installed and you are signed in.
- VS Code 1.93.0 or later is required for the Chat Participant API.
- Try **Ctrl+Shift+P** → **Developer: Reload Window**.

### SQLite unavailable — Cannot find module 'bindings'

**Symptom in Output channel:**
```
[WARN]  SQLite unavailable — running without persistence.
        Cannot find module 'bindings'
```

**Cause:** The `.vsix` was packaged with `--no-dependencies`, which silently omits all of `node_modules` — including the `bindings` and `file-uri-to-path` packages that `better-sqlite3` needs at runtime.

**Fix:** Repackage without the flag:
```bash
cd C:\dev\Roadie\roadie
npx @vscode/vsce package    # no --no-dependencies
code --install-extension roadie-0.5.0.vsix --force
```

---

### SQLite unavailable — ABI / version mismatch

**Symptom in Output channel:**
```
[WARN]  SQLite unavailable — running without persistence.
        The module 'better_sqlite3.node' was compiled against a different Node.js version
        using NODE_MODULE_VERSION 115. This version of Node.js requires NODE_MODULE_VERSION 125.
```
or
```
        'CopyablePersistentTraits' is not a member of 'v8'
```

**Cause:** The `better-sqlite3` binary was compiled for the wrong Electron/V8 version.
- `NODE_MODULE_VERSION` mismatch → compiled for Node.js 20 (standalone), but VS Code's Electron uses a different ABI.
- `CopyablePersistentTraits` error → `better-sqlite3 v9.x` is installed; it uses deprecated V8 APIs removed in Electron 35+. **v12+ is required for VS Code 1.100+ (Electron 35+).**

**Fix — upgrade and recompile:**

```bash
cd C:\dev\Roadie\roadie

# Step 1: Find VS Code's Electron version
# Open VS Code → Help → About, note the Electron version (e.g. 39.8.5)
# Or read it directly:
#   C:\Users\<you>\AppData\Local\Programs\Microsoft VS Code\<hash>\resources\app\package.json
#   Look for: "electron": "39.8.5"

# Step 2: Upgrade better-sqlite3 (v12+ required for Electron 35+)
npm install better-sqlite3@latest

# Step 3: Compile the binary against VS Code's Electron headers
npm rebuild better-sqlite3 --runtime=electron --target=39.8.5 --dist-url=https://electronjs.org/headers
# Replace 39.8.5 with your actual Electron version

# Step 4: Rebuild extension and repackage
npm run build
npx @vscode/vsce package
code --install-extension roadie-0.5.0.vsix --force
```

After reload, the Output channel should show:
```
[INFO]  SQLite persistence initialised — ...project-model.db (0 existing records, workflowHistory=false)
```

---

## Electron ABI Compatibility

This section is required reading before packaging for distribution.

`better-sqlite3` is a native module. The compiled `.node` binary is tied to a specific V8/Node.js ABI. VS Code's extension host runs inside Electron, which ships its own Node.js — different ABI from standalone Node.js.

**Verified working configuration:**

| VS Code | Electron | better-sqlite3 |
|---|---|---|
| 1.115.0 | 39.8.5 | v12.9.0, compiled for Electron 39.8.5 |

**Rule:** Every time you upgrade VS Code on the packaging machine, re-run the rebuild command with the new Electron version. The binary in `node_modules` will not automatically update.

**The complete dependency chain inside the .vsix:**

| File | Role |
|---|---|
| `node_modules/better-sqlite3/build/Release/better_sqlite3.node` | Native SQLite binary (must match Electron ABI) |
| `node_modules/better-sqlite3/lib/**` | JavaScript wrapper |
| `node_modules/bindings/bindings.js` | Finds and loads the `.node` file at runtime |
| `node_modules/file-uri-to-path/index.js` | `bindings` dependency — converts `file://` URIs on Windows |

All four components are required. Missing any one produces `Cannot find module` at activation time. All four are included via the `.vscodeignore` `!` re-inclusion rules — see `02_Technical_Architecture/Extension Manifest & Configuration.md` for the complete `.vscodeignore`.
