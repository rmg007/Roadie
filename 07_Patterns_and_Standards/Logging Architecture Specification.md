# Logging Architecture Specification

## Overview

The Roadie extension routes all diagnostic output through a single logging subsystem defined in `src/shell/logger.ts`. Before this subsystem existed the extension had no persistent logs — failures were silent and debugging required adding temporary `console.log` calls. The design provides a VS Code Output channel named **Roadie** that captures every lifecycle event, warning, and error from activation through deactivation, in a format that is immediately viewable without opening a debugger.

---

## Module Identity

| Property | Value |
|---|---|
| Source file | `src/shell/logger.ts` |
| Depends on | `vscode` (OutputChannel API) |
| Used by | All modules (via `getLogger()`) |
| Exports | `Logger`, `RoadieLogger`, `NullLogger`, `initLogger()`, `getLogger()` |

---

## The Logger Interface

All consumer code imports only the `Logger` interface, never the concrete class. This keeps every module decoupled from the implementation and makes the logger swappable (e.g. for a `NullLogger` in tests).

```typescript
export interface Logger {
  info(msg: string): void;
  warn(msg: string, err?: unknown): void;
  error(msg: string, err?: unknown): void;
  debug(msg: string): void;
}
```

- `info` — key lifecycle events
- `warn` — recoverable issues; accepts an optional `err` value for context
- `error` — unrecoverable failures; accepts an optional `err` value for full stack traces
- `debug` — verbose diagnostics; safe to leave in production code

---

## RoadieLogger — The Real Implementation

`RoadieLogger` is the concrete logger used in production. It implements both `Logger` and `vscode.Disposable`.

### Output Channel

On construction, `RoadieLogger` creates a VS Code OutputChannel:

```typescript
const outputChannel = vscode.window.createOutputChannel('Roadie');
```

This channel appears in VS Code's Output panel under the name **Roadie**. It persists for the full extension lifetime and is visible to the developer without attaching a debugger.

### Timestamp Format

Every log line is prefixed with a wall-clock timestamp derived from:

```typescript
new Date().toISOString().replace('T', ' ').slice(0, 23)
```

This produces the format `YYYY-MM-DD HH:MM:SS.mmm`, for example:

```
2026-04-14 09:23:45.123
```

### Log Line Format

```
YYYY-MM-DD HH:MM:SS.mmm [LEVEL] message
```

Level prefixes:

| Method | Prefix |
|---|---|
| `info()` | `[INFO]` |
| `warn()` | `[WARN]` |
| `error()` | `[ERROR]` |
| `debug()` | `[DEBUG]` |

### Error Formatting

`warn()` and `error()` accept an optional second argument of type `unknown`. When provided:

- If the value is an `Error` instance: appends `err.message` followed by `err.stack`
- Otherwise: appends `String(err)`

This ensures that both typed `Error` objects and raw thrown values (strings, numbers, plain objects) are always serialised to something readable.

### Disposal

`dispose()` forwards to `outputChannel.dispose()`, releasing the VS Code resource when the extension deactivates.

---

## NullLogger — The Safe Fallback

`NullLogger` implements `Logger` with no-op methods. It is the default value of the module-level `_logger` variable, meaning that any module calling `getLogger()` before `initLogger()` runs will silently discard log calls rather than throwing a `ReferenceError`.

This matters in two situations:

1. **Early startup** — modules may be imported and partially initialised before `activate()` completes
2. **Test environments** — unit tests that import modules with logging side-effects do not need to bootstrap a real VS Code environment

`NullLogger` has no constructor arguments and no state.

---

## Singleton Pattern

```typescript
let _logger: Logger = new NullLogger(); // safe default until activate() runs

export function initLogger(): RoadieLogger {
  const l = new RoadieLogger();
  _logger = l;
  return l;
}

export function getLogger(): Logger {
  return _logger;
}
```

Key properties of this design:

- **No import cycles.** Any module can call `getLogger()` without importing any other Roadie module. The logger has no dependencies on the rest of the codebase.
- **No thread-safety concerns.** Node.js (the VS Code extension host runtime) is single-threaded. The assignment `_logger = l` is atomic.
- **Typed return from `initLogger()`.** The function returns the concrete `RoadieLogger` so that `activate()` can register it with the DI container as a `Disposable`. Callers that only need to log use `getLogger()` and receive only the `Logger` interface.

---

## Lifecycle

### Initialisation in `activate()`

`initLogger()` MUST be called as the very first statement inside `activate()` in `extension.ts`, before any other initialisation code runs. The returned instance is immediately registered with the DI container so that VS Code disposes it cleanly when the extension deactivates.

```typescript
// extension.ts
export async function activate(context: vscode.ExtensionContext): Promise<void> {
  // Step 1 — logger must come first, always
  const logger = initLogger();
  container.register(logger); // registers as Disposable

  // All subsequent initialisation can now call getLogger() safely
  const log = getLogger();
  log.info('Roadie extension activating...');

  // ... rest of activate()
}
```

If any code runs before `initLogger()` and calls `getLogger()`, it receives the `NullLogger` — log calls are silently discarded. This is safe but means those early messages are lost. To avoid missing activation messages, no initialisation logic should precede `initLogger()`.

### Deactivation

When VS Code deactivates the extension it calls `dispose()` on every registered `Disposable`. Because `RoadieLogger` is registered with the DI container, `outputChannel.dispose()` is called automatically. No explicit teardown is needed in `deactivate()`.

---

## Usage Pattern

Every module that needs logging follows the same two-line pattern:

```typescript
import { getLogger } from '../shell/logger';

// Inside any function or class method:
const log = getLogger();
log.info('Starting analysis...');
log.warn('Workspace folder not found — skipping', err);
log.error('Startup analysis failed', err);
log.debug('File skipped (content unchanged): src/foo.ts');
```

There is no need to pass a logger through constructors or function parameters. `getLogger()` is a module-level function that always returns the current singleton — `NullLogger` before activation, `RoadieLogger` after.

---

## How to View Logs

1. Open VS Code with the Roadie extension active
2. Open the Output panel: **View → Output** (or `Ctrl+Shift+U` / `Cmd+Shift+U`)
3. Click the dropdown on the right side of the Output panel
4. Select **Roadie** from the list
5. All `info`, `warn`, `error`, and `debug` messages from activation onwards appear here in chronological order

The channel is append-only and scrolls automatically. It is not cleared between commands — the full session history is visible until the extension deactivates or VS Code is closed.

---

## Log Level Guidelines

| Level | When to use | Examples |
|---|---|---|
| `info` | Key lifecycle events that mark progress or completion | Extension activated, analysis complete, file written, command invoked |
| `warn` | Recoverable issues where execution continues but something is degraded | SQLite unavailable, no workspace folder found, setting saved but service not ready |
| `error` | Unrecoverable failures — always pass the full `Error` object | Startup analysis failed, agent spawn failed, file write rejected |
| `debug` | Verbose detail useful only when actively diagnosing a problem | Files skipped as unchanged, step timing, token counts, intermediate state |

`debug` calls are always active — there is no runtime log-level filter. Keep `debug` messages concise enough that a full session log remains readable.

---

## Modules That Use Logging

| Module | What it logs |
|---|---|
| `extension.ts` | Activation/deactivation lifecycle, SQLite init result, startup analysis trigger |
| `shell/chat-participant.ts` | Intent classification result, workflow start/end/status |
| `engine/workflow-engine.ts` | Every state transition, step start and end with timing |
| `engine/step-executor.ts` | Each attempt, escalation events, timeout detection |
| `spawner/agent-spawner.ts` | Model resolution, prompt size, response size, spawn errors |
| `analyzer/project-analyzer.ts` | Analysis start and finish with file/symbol counts |
| `generator/file-generator.ts` | Each file written (with KB size) or skipped (unchanged) |
| `shell/commands.ts` | Each VS Code command invocation |

---

## Full Implementation

```typescript
// src/shell/logger.ts

/**
 * @module logger
 * @description Singleton logging subsystem for the Roadie extension.
 *   Writes timestamped lines to a VS Code OutputChannel named 'Roadie'.
 *   Safe to call from any module at any time via getLogger().
 * @inputs Log messages (string) and optional Error values
 * @outputs VS Code Output panel entries (View → Output → Roadie)
 * @depends-on vscode (OutputChannel API)
 * @depended-on-by All modules
 */

import * as vscode from 'vscode';

// ─── Logger interface ────────────────────────────────────────────────────────

export interface Logger {
  info(msg: string): void;
  warn(msg: string, err?: unknown): void;
  error(msg: string, err?: unknown): void;
  debug(msg: string): void;
}

// ─── NullLogger — safe no-op fallback ────────────────────────────────────────

export class NullLogger implements Logger {
  info(_msg: string): void {}
  warn(_msg: string, _err?: unknown): void {}
  error(_msg: string, _err?: unknown): void {}
  debug(_msg: string): void {}
}

// ─── RoadieLogger — real implementation ──────────────────────────────────────

export class RoadieLogger implements Logger, vscode.Disposable {
  private readonly outputChannel: vscode.OutputChannel;

  constructor() {
    this.outputChannel = vscode.window.createOutputChannel('Roadie');
  }

  private timestamp(): string {
    return new Date().toISOString().replace('T', ' ').slice(0, 23);
  }

  private formatError(err: unknown): string {
    if (err instanceof Error) {
      return `${err.message}\n${err.stack ?? ''}`;
    }
    return String(err);
  }

  info(msg: string): void {
    this.outputChannel.appendLine(`${this.timestamp()} [INFO]  ${msg}`);
  }

  warn(msg: string, err?: unknown): void {
    const suffix = err !== undefined ? `\n${this.formatError(err)}` : '';
    this.outputChannel.appendLine(`${this.timestamp()} [WARN]  ${msg}${suffix}`);
  }

  error(msg: string, err?: unknown): void {
    const suffix = err !== undefined ? `\n${this.formatError(err)}` : '';
    this.outputChannel.appendLine(`${this.timestamp()} [ERROR] ${msg}${suffix}`);
  }

  debug(msg: string): void {
    this.outputChannel.appendLine(`${this.timestamp()} [DEBUG] ${msg}`);
  }

  dispose(): void {
    this.outputChannel.dispose();
  }
}

// ─── Singleton ────────────────────────────────────────────────────────────────

let _logger: Logger = new NullLogger();

/**
 * Creates a RoadieLogger, installs it as the singleton, and returns it.
 * MUST be called as the first statement in activate() before any other
 * initialisation. Register the returned instance with the DI container
 * so it is disposed when the extension deactivates.
 */
export function initLogger(): RoadieLogger {
  const l = new RoadieLogger();
  _logger = l;
  return l;
}

/**
 * Returns the current logger singleton.
 * Safe to call at any time — returns NullLogger before initLogger() runs.
 */
export function getLogger(): Logger {
  return _logger;
}
```

---

## Design Rationale

**VS Code OutputChannel** is the standard mechanism for persistent, developer-visible logs in VS Code extensions. Unlike `console.log`, OutputChannel output is not mixed with the browser devtools console and remains readable without attaching a debugger or running the Extension Development Host in verbose mode.

**Singleton via module-level variable** avoids the need to pass a logger through every constructor and function signature. There are no circular import risks because `logger.ts` depends only on `vscode` and nothing else in the Roadie codebase.

**NullLogger as the default** means that any module importing and calling `getLogger()` before `activate()` completes does not crash. This is important because TypeScript module initialisation (top-level code, class field initialisers) runs before `activate()` is called by VS Code.

**Returning `RoadieLogger` from `initLogger()`** (rather than `Logger`) gives `activate()` access to the concrete type, which is needed to register it as a `vscode.Disposable` with the DI container. All other callers receive the narrower `Logger` interface and cannot accidentally call `dispose()` themselves.
