# 🔐 Security & Migration Specification

## Extension Security Model

Roadie is a VS Code extension that operates with **workspace trust** semantics and the **VS Code extension host process** security boundary.

---

## Permission Model

### What Roadie CAN Do

| Permission | API Used | Constraint |
|---|---|---|
| Read workspace files | `vscode.workspace.fs.readFile()` | Workspace-scoped only |
| Write workspace files | `vscode.workspace.applyEdit()` | Workspace-scoped only, all writes must be user-visible applyEdit (undoable) |
| Execute shell commands | `child_process.exec()` or `vscode.Terminal` | Only allowlisted commands (see below) |
| Read VS Code configuration | `vscode.workspace.getConfiguration('roadie')` | `roadie.*` namespace only |
| Create SQLite database | `better-sqlite3` | Must write to `.github/.roadie/` only |
| Use LLM APIs | `vscode.lm.selectChatModels()` | Via VS Code LM API only — no direct HTTP calls to LLM providers |
| Watch file system | `vscode.workspace.createFileSystemWatcher()` | Workspace-scoped glob patterns only |
| Stream chat responses | `vscode.ChatResponseStream` | Only within a `ChatParticipant.requestHandler` scope |

### What Roadie CANNOT Do

- Make direct HTTP requests to external services (no `fetch`, no `axios`)
- Access files outside the current workspace
- Access environment variables beyond what VS Code surfaces
- Spawn elevated/administrator processes
- Persist data outside `.github/.roadie/` and VS Code's extension storage (`ExtensionContext.globalStorageUri`)

---

## Shell Command Allowlist

Roadie executes shell commands only for running project tests and package manager operations (Dependency workflow). All commands must match this allowlist before execution:

```ts
const ALLOWED_SHELL_COMMANDS: RegExp[] = [
  // Test runners
  /^npm (run )?test(\s+.*)?$/,
  /^npx jest(\s+.*)?$/,
  /^npx vitest(\s+.*)?$/,
  /^npx mocha(\s+.*)?$/,
  /^yarn test(\s+.*)?$/,
  /^pnpm test(\s+.*)?$/,

  // Package manager — install specific package only (no rm, no scripts)
  /^npm install [\w\-@\/\^~.]+(\s[\w\-@\/\^~.]+)*$/,
  /^yarn add [\w\-@\/\^~.]+(\s[\w\-@\/\^~.]+)*$/,
  /^pnpm add [\w\-@\/\^~.]+(\s[\w\-@\/\^~.]+)*$/,
];

function isCommandAllowed(command: string): boolean {
  return ALLOWED_SHELL_COMMANDS.some(pattern => pattern.test(command.trim()));
}
```

**Enforcement point:** `StepExecutor.executeShellCommand()` calls `isCommandAllowed()` before invoking `child_process`. If the command does not match, it throws `RoadieError` with `category: 'validation'` and does NOT execute.

---

## SQLite Database Security

### File Location

```
{workspaceRoot}/.github/.roadie/project-model.db
```

The `.github/.roadie/` directory is committed with a `.gitignore` file that excludes `project-model.db`:

```gitignore
# Roadie project model (machine-local, not for version control)
project-model.db
```

### File Permissions

When creating `.github/.roadie/project-model.db`, Roadie must set POSIX file permissions to `0600` (owner read/write only) immediately after creation:

```ts
import { chmodSync } from 'fs';

function openDatabase(dbPath: string): Database {
  const db = new Database(dbPath);
  chmodSync(dbPath, 0o600);          // owner r/w only — no group/world access
  applyOpenTimePragmas(db);
  runMigrations(db);
  return db;
}
```

> **Windows note:** `chmodSync` is a no-op on Windows. The file inherits NTFS ACLs from the `.github/.roadie/` directory, which is created with default user-only ACLs by `fs.mkdirSync`. Acceptable — the spec requirement is POSIX-only.

---

## SQLite Database Pragmas

### Mandatory Open-Time Pragmas

These pragmas **must** be applied on every database open, before any other statement. Order is significant: `journal_mode` before `busy_timeout`.

```ts
// src/infrastructure/database.ts

function applyOpenTimePragmas(db: Database): void {
  // WAL mode: enables concurrent read + one write; prevents read-blocking writes
  db.pragma('journal_mode = WAL');

  // Retry for up to 5 seconds before throwing SQLITE_BUSY
  // Phase 2 MCP server and the extension share the same file — this is mandatory
  db.pragma('busy_timeout = 5000');

  // Enforce FK constraints (better-sqlite3 has this OFF by default)
  db.pragma('foreign_keys = ON');
}
```

**Why WAL is mandatory:** Phase 2 introduces an MCP server process that reads the same `.github/.roadie/project-model.db` file while the extension is writing. Without WAL, any simultaneous access produces `SQLITE_BUSY`. With WAL, readers never block writers.

**Why busy_timeout = 5000 is mandatory:** Even with WAL, write serialization is still required. 5000ms gives the VS Code UI enough time to remain responsive without surfacing an error to the user on the first write collision.

---

## Schema Version Table

Roadie uses an explicit `schema_version` table instead of relying solely on `PRAGMA user_version`, which is not preserved by all SQLite backup tools.

```sql
-- Created by Migration 001 — must be the first statement in that migration
CREATE TABLE IF NOT EXISTS schema_version (
  version    INTEGER PRIMARY KEY NOT NULL,
  applied_at TEXT    NOT NULL     -- ISO 8601 timestamp, e.g. '2024-11-01T12:00:00.000Z'
);
```

Both `schema_version` and `PRAGMA user_version` are updated atomically inside each migration transaction:

```ts
db.transaction(() => {
  db.exec(MIGRATIONS[v]);
  db.prepare('INSERT INTO schema_version (version, applied_at) VALUES (?, ?)').run(v, new Date().toISOString());
  db.pragma(`user_version = ${v}`);
})();
```

---

## Schema Migrations

### Migration 001 — Full DDL

```sql
-- Migration 001: Initial schema
CREATE TABLE IF NOT EXISTS schema_version (
  version    INTEGER PRIMARY KEY NOT NULL,
  applied_at TEXT    NOT NULL
);

CREATE TABLE IF NOT EXISTS tech_stack (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT    NOT NULL,
  version     TEXT,
  confidence  REAL    NOT NULL DEFAULT 1.0,
  source_file TEXT    NOT NULL,
  detected_at TEXT    NOT NULL  -- ISO 8601
);

CREATE TABLE IF NOT EXISTS directories (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  path        TEXT    NOT NULL UNIQUE,
  role        TEXT    NOT NULL,  -- 'source' | 'test' | 'config' | 'docs' | 'build'
  file_count  INTEGER NOT NULL DEFAULT 0,
  scanned_at  TEXT    NOT NULL   -- ISO 8601
);

CREATE TABLE IF NOT EXISTS patterns (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  pattern     TEXT    NOT NULL,
  description TEXT    NOT NULL,
  weight      REAL    NOT NULL DEFAULT 1.0
);

CREATE TABLE IF NOT EXISTS commands (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  type        TEXT    NOT NULL,  -- 'test' | 'build' | 'lint' | 'dev'
  command     TEXT    NOT NULL,
  confidence  REAL    NOT NULL DEFAULT 1.0
);
```

### Migration 002 (Phase 1.5)

```sql
-- Migration 002: Edit tracking (Phase 1.5 only — stub in Phase 1)
CREATE TABLE IF NOT EXISTS edit_records (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path    TEXT    NOT NULL,
  workflow_id  TEXT    NOT NULL,
  diff_text    TEXT    NOT NULL,
  recorded_at  TEXT    NOT NULL  -- ISO 8601
);
```

### Migration Algorithm

```ts
function runMigrations(db: Database): void {
  // 1. Read current version
  const row = db.prepare('PRAGMA user_version').get() as { user_version: number };
  let currentVersion = row.user_version;

  // 2. Apply all pending migrations in order
  const targetVersion = Math.max(...Object.keys(MIGRATIONS).map(Number));

  for (let v = currentVersion + 1; v <= targetVersion; v++) {
    if (MIGRATIONS[v]) {
      db.transaction(() => {
        db.exec(MIGRATIONS[v]);
        db.prepare('INSERT INTO schema_version (version, applied_at) VALUES (?, ?)').run(v, new Date().toISOString());
        db.pragma(`user_version = ${v}`);
        logger.info(`[DB] Migration applied: v${v}`);
      })();
    }
  }
}
```

**Rules:**
- Never drop columns in a migration — add nullable columns instead.
- Never rename columns — add a new column and copy data.
- If a migration throws, log the error and disable Roadie for this session (do not crash VS Code).

---

## SQLITE_CORRUPT Recovery

If `better-sqlite3` throws with `error.code === 'SQLITE_CORRUPT'`, Roadie must not crash and must not leave the user without a working state. Recovery procedure:

```ts
// src/infrastructure/database.ts

import { renameSync, existsSync } from 'fs';
import { join } from 'path';

function openWithCorruptionRecovery(dbPath: string): Database {
  try {
    return openDatabase(dbPath);
  } catch (error: unknown) {
    if (
      error instanceof Error &&
      'code' in error &&
      (error as NodeJS.ErrnoException).code === 'SQLITE_CORRUPT'
    ) {
      // 1. Rename corrupted file — preserve for diagnostic / user recovery
      const suffix = new Date().toISOString().replace(/[:.]/g, '-');
      const corruptPath = `${dbPath}.corrupted.${suffix}`;
      renameSync(dbPath, corruptPath);
      logger.warn(`[DB] Corrupt database renamed to ${corruptPath}. Rebuilding from filesystem.`);

      // 2. Open a fresh database (migration 001 will run on next call)
      const freshDb = openDatabase(dbPath);

      // 3. Notify user — they should re-run @roadie init or trigger rescan
      vscode.window.showWarningMessage(
        'Roadie: Project database was corrupted and has been rebuilt. Run "Roadie: Rescan Project" to restore context.',
        'Rescan Now',
      ).then(choice => {
        if (choice === 'Rescan Now') {
          vscode.commands.executeCommand('roadie.rescan');
        }
      });

      return freshDb;
    }
    throw error; // Re-throw non-corruption errors
  }
}
```

**Recovery invariant:** After `openWithCorruptionRecovery`, the returned `Database` instance is always a clean, freshly-migrated database. The corrupted file is never deleted automatically — only renamed, so the developer can inspect it.

---

## Schema Migrations (Summary)

Database migrations are **forward-only**. There is no rollback mechanism. Migration files are embedded in `database.ts` as versioned SQL strings (see full DDL above):

```ts
const MIGRATIONS: Record<number, string> = {
  1: `/* see Migration 001 full DDL above */`,
  2: `/* see Migration 002 DDL above */`,
  // Future migrations: increment version number, add SQL
};
```

**Migration run trigger:** Runs on every extension activation, inside `openWithCorruptionRecovery`.

---


## Workspace Trust



Roadie does not activate in untrusted workspaces. The `package.json` manifest must include:

```json
{
  "capabilities": {
    "untrustedWorkspaces": {
      "supported": "limited",
      "description": "Roadie requires workspace trust to read project files and run test commands."
    }
  }
}
```

In limited mode (untrusted workspace):
- The LLM enrichment features are disabled.
- Roadie responds with: "This workspace is not trusted. Roadie requires a trusted workspace to operate."
- The status bar appears but shows "Limited Mode" tooltip.

---

## LLM Output Sanitization

All LLM output that writes to the filesystem goes through sanitization before `vscode.workspace.applyEdit()`:

```ts
function sanitizeFileContent(raw: string, filePath: string): string {
  // 1. Strip markdown code fences if LLM wrapped the file content
  const fencePattern = /^```[\w]*\n([\s\S]*?)\n```$/;
  const match = raw.match(fencePattern);
  const content = match ? match[1] : raw;

  // 2. Enforce max file size (500KB)
  if (Buffer.byteLength(content, 'utf8') > 500_000) {
    throw new RoadieError({
      code: 'FILE_TOO_LARGE',
      category: 'validation',
      userFacing: true,
      message: `Generated content for ${filePath} exceeds 500KB limit.`,
    });
  }

  // 3. For .md files: no executable script tags
  if (filePath.endsWith('.md')) {
    const scriptTagPattern = /<script[\s\S]*?>[\s\S]*?<\/script>/gi;
    return content.replace(scriptTagPattern, '<!-- roadie:removed:script-tag -->');
  }

  return content;
}
```

---

## Token Budget Enforcement

Each workflow step enforces a per-step token budget:

| Step Type | Max Input Tokens | Max Output Tokens |
|---|---|---|
| Classification (Tier 0) | 2,000 | 200 |
| Research / Diagnosis | 8,000 | 2,000 |
| Code Generation | 12,000 | 4,000 |
| Code Review (parallel pass) | 8,000 | 2,000 |
| Documentation | 10,000 | 3,000 |
| Summary | 5,000 | 1,000 |

These are enforced in `AgentSpawner.spawn()`:

```ts
const tokenOptions: vscode.LanguageModelChatRequestOptions = {
  maxOutputTokens: BUDGET_PER_STEP[step.type].maxOutput,
};
```

Inputs that exceed the budget are truncated by `PromptBuilder.build()` using the following priority order:

1. Core task prompt (never truncated)
2. Tech stack context (truncated last)
3. Directory structure (truncated second-to-last)
4. Prior step outputs (truncated first)

---

## Extension Storage

Two storage locations are used:

| Location | API | Contents | Cleared On |
|---|---|---|---|
| `.github/.roadie/project-model.db` | `better-sqlite3` | Project model cache | Manual (`roadie.reset` command) |
| `ExtensionContext.globalStorageUri` | `vscode.ExtensionContext` | User preferences, session metadata | Extension uninstall |
| `ExtensionContext.workspaceStorageUri` | `vscode.ExtensionContext` | Workspace-specific state (last scan timestamp) | Workspace close |

**Rule:** Never write to `os.tmpdir()` or any path outside these two locations.

---

## Secrets — What Roadie Does NOT Store

Roadie does **not** store, read, or transmit:
- Personal GitHub tokens
- LLM API keys (the VS Code LM API handles auth transparently)
- User email or identity
- Any telemetry data (telemetry is off by default; controlled by `roadie.telemetry` setting)

### ROADIE_API_KEY — Not Used in Phase 1

Roadie Phase 1 does **not** use any external API key. All LLM access is through the VS Code Language Model API (`vscode.lm.selectChatModels()`), which delegates auth to the user's GitHub Copilot subscription.

If a Phase 2 feature requires direct API access (e.g., a cloud-hosted inference endpoint), the only acceptable env var name is `ROADIE_API_KEY`. Rules:

- **Never read from `.mcp.json`** — `.mcp.json` is a project-level file that may be committed to version control. Storing API keys in it is a critical secret-exposure vulnerability.
- **Read from `process.env.ROADIE_API_KEY`** only, and only from the extension host process (not from webview context).
- **Never log the value** — log only whether the key is set: `logger.info('API key present:', !!process.env.ROADIE_API_KEY)`.
- **The SQLite database must never store the key** — `.github/.roadie/project-model.db` is machine-local but not encrypted. Keys written here are readable by any process with workspace filesystem access.

```ts
// ✅ Correct — Phase 2 example
function getApiKey(): string {
  const key = process.env.ROADIE_API_KEY;
  if (!key) {
    throw new RoadieError({
      code: 'MISSING_API_KEY',
      category: 'validation',
      userFacing: true,
      message: 'ROADIE_API_KEY environment variable is not set. Set it in your shell profile, not in .mcp.json.',
    });
  }
  return key;
}

// ❌ Never do this
const key = JSON.parse(fs.readFileSync('.mcp.json', 'utf8')).roadieApiKey;
```

---



---

## Phase Migration Plan (Phase 1 → Phase 2)

When Phase 2 modules are added, the following migration steps apply:

| Migration | Trigger | Action |
|---|---|---|
| DB schema v2 | Phase 2 activation | Add `confidence` column to `tech_stack` table via migration v2 |
| File Watcher activation | Phase 1.5 module added | Register `createFileSystemWatcher` on extension activate; deregister on deactivate |
| New intent type added | Phase 2 classification update | Add new intent to `IntentType` union in `types.ts`, add patterns in `intent-patterns.ts`, add workflow definition file |

**Migration rule:** All Phase 1 → Phase 1.5 → Phase 2 transitions must be backward-compatible. A VS Code extension that was activated by Phase 1 code should initialize cleanly after Phase 1.5 database migrations.
