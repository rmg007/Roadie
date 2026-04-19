# IDE Detector Specification

**Module:** `detector`  
**File:** `src/detector/ide-detector.ts`  
**Version:** v1.0.0 (shipped 2026-04-17)  
**Status:** Stable

---

## Overview

The IDE Detector module detects which IDEs and tools are active in the current workspace. It uses environment variables (fast, reliable) and file system markers (comprehensive) to identify VS Code, Cursor, Claude Code, and Windsurf. It also detects when Roadie is running inside a Claude Code hook session.

The detector is consumed by the File Generator (`src/generator/file-generator.ts`) and cached per session for performance.

---

## Public Interface

### `DetectionResult`

```typescript
export interface DetectionResult {
  isVSCode: boolean;      // true if VSCODE_PID env var is set
  isCursor: boolean;      // true if .cursor/ directory exists in workspace root
  isClaudeCode: boolean;  // true if .mcp.json or .claude/ exists in workspace root
  isWindsurf: boolean;    // true if .windsurf/ directory exists in workspace root
  detectedIDEs: string[]; // all detected IDE keys: 'vscode', 'cursor', 'claude-code', 'windsurf'
  primaryIDE: string | null; // set only when exactly one IDE detected; null when ambiguous
}
```

### `detectIDEs(workspaceRoot: string): Promise<DetectionResult>`

Async function. Detects IDEs active in the workspace identified by `workspaceRoot`.

**Signature:**
```typescript
export async function detectIDEs(workspaceRoot: string): Promise<DetectionResult>
```

**Detection order:**
1. **VS Code** — checks `process.env.VSCODE_PID` (set by VS Code when running extensions).
2. **Cursor** — checks for `.cursor/` directory in `workspaceRoot` via `fs.stat`.
3. **Claude Code** — checks for `.mcp.json` OR `.claude/` in `workspaceRoot`.
4. **Windsurf** — checks for `.windsurf/` directory in `workspaceRoot` via `fs.stat`.

**`primaryIDE` logic:** Set to the single detected IDE key if exactly one IDE is detected; `null` if zero or more than one detected.

**Never throws** — all file system errors are caught internally. Returns a `DetectionResult` with all `false` booleans if detection fails entirely.

### `isRunningUnderClaudeCodeHooks(): boolean`

Synchronous function. Returns `true` if Roadie is running inside a Claude Code hook session (i.e., invoked as a hook callback, not as a VS Code extension).

**Signature:**
```typescript
export function isRunningUnderClaudeCodeHooks(): boolean
```

**Detection:** Checks for the presence of any of these environment variables set by Claude Code hooks:
- `TOOL_USE_ID`
- `TRANSCRIPT_PATH`
- `TOOL`
- `FILE`

Returns `true` if at least one is set and non-empty.

---

## Supported IDEs

| IDE | Detection Method | Marker |
|---|---|---|
| VS Code | Environment variable | `VSCODE_PID` |
| Cursor | File system | `.cursor/` directory in workspace root |
| Claude Code | File system | `.mcp.json` OR `.claude/` directory in workspace root |
| Windsurf | File system | `.windsurf/` directory in workspace root |

---

## Usage Example

```typescript
import { detectIDEs, isRunningUnderClaudeCodeHooks } from './detector/ide-detector';

const result = await detectIDEs(workspaceRoot);
// result.isVSCode  → true if running as VS Code extension
// result.isCursor  → true if .cursor/ found
// result.primaryIDE → 'vscode' | 'cursor' | 'claude-code' | 'windsurf' | null

if (isRunningUnderClaudeCodeHooks()) {
  // Running as a Claude Code hook — adjust behavior accordingly
}
```

---

## Performance

Detection is O(1) for env-var check and O(fs) for each file marker. Total expected latency: < 5 ms per call. The File Generator caches the `DetectionResult` per session — `detectIDEs()` is called once at startup and again only if the workspace root changes.

---

## Phase 2 Notes

In Phase 2 (v1.1+), detection results will be used to:
- Conditionally generate tool-specific config files (`.mcp.json` for Claude Code, Windsurf-specific rules, etc.)
- Emit tool-specific hints in generated files (e.g., Claude Code hooks inline in `CLAUDE.md`)
- Support native Windsurf and LSP-based detection

For v1.0.0, detection results are available but do not gate file generation — all files are generated unconditionally.

---

## Related Files

- `src/detector/ide-detector.ts` — Implementation
- `src/generator/file-generator.ts` — Primary consumer
- `roadie_docs/IDE_DETECTION_PROPOSAL.md` — Original proposal (now implemented; see note at top)
- `roadie_docs/FILE_GENERATION_STRATEGY.md` — Why generation is static in v1.0.0
