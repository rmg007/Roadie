# Public API Specification

**Module:** `api`  
**File:** `src/api/index.ts`  
**Version:** v1.0.0 (shipped 2026-04-17)  
**Status:** Stable (semver-stable surface)

---

## Overview

`src/api/index.ts` is the **stable public API surface** for Roadie v1.0.0. Everything exported from this file is semver-stable — callers (extension contributors, test authors, external integrations) may depend on these exports without risk of breakage within the v1.x series. Everything NOT exported from this file is `@internal` and subject to change without notice.

---

## Exports

### `ClassificationResult` (type)

Re-exported from `src/types.ts`.

```typescript
export type { ClassificationResult } from '../types';
```

Represents the output of intent classification — the result of routing a user message to one of the 8 intent types.

**Usage:** Type-annotate variables holding classifier output; pass to workflow engine.

---

### `IntentClassifier` (class)

Re-exported from `src/classifier/intent-classifier.ts`.

```typescript
export { IntentClassifier } from '../classifier/intent-classifier';
```

The intent classification engine. Takes a raw user message and returns a `ClassificationResult` with an intent type and confidence score.

**Role:** Primary entry point for routing `@roadie` messages to the correct workflow.

---

### `RoadieError` (class)

Re-exported from `src/shell/errors.ts`.

```typescript
export { RoadieError } from '../shell/errors';
```

Base error class for all Roadie-originated errors. Extends `Error` with a `code` field drawn from the closed error taxonomy defined in `08_Integration_and_Testing/Testing, Error Handling & Security.md`.

**Usage:** Catch `RoadieError` in extension hosts to distinguish Roadie errors from unexpected system errors.

---

### `TelemetryReporter` (class)

Re-exported from `src/shell/telemetry.ts`.

```typescript
export { TelemetryReporter } from '../shell/telemetry';
```

Anonymous aggregate telemetry reporter. Sends workflow type and success-rate data (no code, no filenames, no user identifiers) when `roadie.telemetry` is `true`. Extension contributors who build on top of Roadie can inject a custom reporter.

**Usage:** Type-check a custom reporter implementation or extend for contributor telemetry hooks.

---

## Exports Table

| Export | Kind | Source | Role |
|---|---|---|---|
| `ClassificationResult` | `type` | `src/types.ts` | Output type for intent classification |
| `IntentClassifier` | `class` | `src/classifier/intent-classifier.ts` | Classifies user messages into intent types |
| `RoadieError` | `class` | `src/shell/errors.ts` | Base error class with closed error code taxonomy |
| `TelemetryReporter` | `class` | `src/shell/telemetry.ts` | Anonymous aggregate telemetry (opt-in) |

---

## What Is NOT Exported

The following are intentionally internal and not exported from `src/api/index.ts`:

- Workflow engine internals (`WorkflowEngine`, `WorkflowFSM`)
- Agent spawner (`AgentSpawner`)
- Project model (`ProjectModel`, `PersistentProjectModel`)
- File generator internals (`FileGeneratorManager`, `buildFileSpecs`)
- Section manager
- Learning database schema
- All `src/generator/templates/*` files

These may change across minor versions without notice.

---

## Phase 2 Expansion

In Phase 2 (v1.1+), the public API surface is planned to expand with:

- **HTTP server** — REST endpoint exposing project context (`GET /context`, `POST /analyze`)
- **MCP connector** — JSON-RPC `initialize` / `tools/call` server for Claude Code integration
- **OpenAPI schema** — Machine-readable spec for the REST surface

These are not yet implemented. The v1.0.0 surface above is the complete current public API.

---

## Related Files

- `src/api/index.ts` — Implementation (single re-export file)
- `src/types.ts` — `ClassificationResult` definition
- `src/classifier/intent-classifier.ts` — `IntentClassifier` implementation
- `src/shell/errors.ts` — `RoadieError` implementation
- `src/shell/telemetry.ts` — `TelemetryReporter` implementation
- `roadie_docs/02_IDE_Detector_Specification.md` — IDE Detector public API (`detectIDEs`, `isRunningUnderClaudeCodeHooks`)
