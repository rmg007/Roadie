# IDE Detection — Technical Feasibility

> **Note (v1.0.0, 2026-04-17):** This proposal has been implemented. IDE detection shipped in v1.0.0 as `src/detector/ide-detector.ts`. See `02_IDE_Detector_Specification.md` for the canonical specification of the shipped implementation.

**Last Updated:** 2026-04-17

---

## Is It Possible?

**Yes.** Roadie (or any Node.js process) can reliably detect the IDE/tool environment.

---

## Detection Methods (Ranked by Reliability)

### 1. Environment Variables (VS Code ✅)

VS Code sets `VSCODE_PID` when running extensions:

```typescript
function isRunningInVSCode(): boolean {
  return !!process.env.VSCODE_PID;
}
```

**Reliability:** ⭐⭐⭐⭐⭐ (100% in VS Code)  
**Other tools:** Similar env vars exist for other IDEs

| IDE | Environment Variable | Value Example |
|---|---|---|
| VS Code | `VSCODE_PID` | `12345` (process ID) |
| Cursor | `CURSOR_PID` or similar | (likely) |
| Windsurf | Unknown (would need to test) | (unknown) |
| JetBrains | `IDEA_INITIAL_DIRECTORY` | `N/A` (extension model different) |
| Vim/Neovim | Various (LSP-based) | (no single source) |

### 2. File System Markers (IDE Config Detection)

Check for IDE-specific config files in workspace:

```typescript
async function detectIDEsInWorkspace(workspaceRoot: string): Promise<string[]> {
  const detected: string[] = [];
  
  // Cursor config
  if (await fileExists(path.join(workspaceRoot, '.cursor'))) {
    detected.push('cursor');
  }
  
  // Claude Code MCP config
  if (await fileExists(path.join(workspaceRoot, '.mcp.json'))) {
    detected.push('claude-code'); // (uses MCP)
  }
  
  // Claude settings
  if (await fileExists(path.join(workspaceRoot, '.claude'))) {
    detected.push('claude-code'); // (alternative marker)
  }
  
  // Windsurf config
  if (await fileExists(path.join(workspaceRoot, '.windsurf'))) {
    detected.push('windsurf');
  }
  
  // JetBrains IDE
  if (await fileExists(path.join(workspaceRoot, '.idea'))) {
    detected.push('jetbrains');
  }
  
  // LSP-based editors
  if (await fileExists(path.join(workspaceRoot, '.lsp'))) {
    detected.push('lsp-client');
  }
  
  return detected;
}
```

**Reliability:** ⭐⭐⭐⭐ (High, but not definitive—user might have `.mcp.json` without using Claude Code)

### 3. Process Detection (Heavy-Handed)

Check running processes for IDE PIDs:

```typescript
import { execSync } from 'node:child_process';

function detectRunningIDEs(): string[] {
  const detected: string[] = [];
  
  try {
    const processes = execSync('ps aux', { encoding: 'utf8' });
    
    if (processes.includes('code')) detected.push('vscode');
    if (processes.includes('cursor')) detected.push('cursor');
    if (processes.includes('windsurf')) detected.push('windsurf');
    if (processes.includes('java') && processes.includes('.idea')) {
      detected.push('jetbrains');
    }
  } catch {
    // Windows or ps not available
  }
  
  return detected;
}
```

**Reliability:** ⭐⭐⭐ (Moderate; process names vary, requires shell access)  
**Cost:** Spawns subprocess (slow)

### 4. Anthropic SDK / Tool Detection (For Claude Code)

If running under Claude Code hooks, environment variables are set:

```typescript
function isRunningUnderClaudeCode(): boolean {
  // Claude Code sets these when firing hooks
  return !!(
    process.env.TOOL_USE_ID ||
    process.env.TRANSCRIPT_PATH ||
    process.env.TOOL ||
    process.env.FILE
  );
}
```

**Reliability:** ⭐⭐⭐⭐ (High, but only when hooks fire)  
**Context:** Only available during/after hook execution

---

## Proposed Implementation

### Shipped in v1.0.0 (was: Quick Win for v0.7.10)

Add `src/detector/ide-detector.ts`:

```typescript
/**
 * @module ide-detector
 * @description Detects which IDEs/tools are active in the workspace.
 *   Used to conditionally generate tool-specific config files.
 */

export interface DetectionResult {
  isVSCode: boolean;
  isCursor: boolean;
  isClaudeCode: boolean;
  isWindsurf: boolean;
  detectedIDEs: string[]; // All detected
  primaryIDE: string | null; // Most likely (if unambiguous)
}

export async function detectIDEs(workspaceRoot: string): Promise<DetectionResult> {
  const detected: string[] = [];
  
  // Check environment
  if (process.env.VSCODE_PID) {
    detected.push('vscode');
  }
  
  // Check for Cursor markers
  if (await fileExists(path.join(workspaceRoot, '.cursor'))) {
    detected.push('cursor');
  }
  
  // Check for Claude Code markers
  const hasClaudeMarkers = 
    await fileExists(path.join(workspaceRoot, '.mcp.json')) ||
    await fileExists(path.join(workspaceRoot, '.claude'));
  if (hasClaudeMarkers) {
    detected.push('claude-code');
  }
  
  // Check for Windsurf markers
  if (await fileExists(path.join(workspaceRoot, '.windsurf'))) {
    detected.push('windsurf');
  }
  
  return {
    isVSCode: detected.includes('vscode'),
    isCursor: detected.includes('cursor'),
    isClaudeCode: detected.includes('claude-code'),
    isWindsurf: detected.includes('windsurf'),
    detectedIDEs: detected,
    primaryIDE: detected.length === 1 ? detected[0] : null,
  };
}
```

### Usage in File Generator

```typescript
// src/generator/file-generator.ts

async generateAll(model: ProjectModel): Promise<GeneratedFile[]> {
  const detection = await detectIDEs(this.workspaceRoot);
  
  // Always generate base files
  const specs = buildBaseFileSpecs(this.learningDb);
  
  // Conditionally add tool-specific files
  if (detection.isCursor) {
    // Already in base, but could optimize
  }
  if (detection.isClaudeCode) {
    specs.push({
      type: 'mcp_config',
      path: '.mcp.json',
      generate: generateMcpConfig,
    });
    // Future: add claude-hooks when Phase 2 ready
  }
  if (detection.isWindsurf) {
    // Future: Windsurf-specific generator
  }
  
  // Generate all specs
  const results: GeneratedFile[] = [];
  for (const spec of specs) {
    const result = await this.generateFile(spec, model);
    results.push(result);
  }
  return results;
}
```

### Settings (Optional)

```json
{
  "roadie.ideDetection": "auto|disabled|manual",
  "roadie.forceGenerateFor": ["copilot", "cursor", "claude"]
}
```

---

## Limitations

### False Positives
- User has `.mcp.json` but doesn't use Claude Code → detected as Claude Code
- User has `.cursor` directory but doesn't use Cursor IDE → detected as Cursor

### False Negatives
- User has Cursor installed but not in the current workspace session → not detected
- IDE runs in a container or remote session → env vars not visible

### Performance
- File system checks: ~milliseconds (negligible)
- Process enumeration: ~100-500ms (slow, not recommended)

---

## Recommendation

**Shipped in v1.0.0 (was: v0.7.10+ before Phase 2):**

1. ✅ Add `ide-detector.ts` with env var + file system detection
2. ✅ Make detection **opt-in** (add setting: `roadie.enableIDEDetection: false` by default)
3. ✅ Log detected IDEs to Output channel for debugging
4. ✅ Prepare file specs to branch on detection results (but still generate base set)

**For Phase 2:**

1. Use detection to conditionally generate `.mcp.json`, `.claude/settings.json`, Windsurf config
2. Optimize per-IDE: suppress unnecessary files, enhance tool-specific files
3. Add `roadie.ideDetection` setting: `"auto"` (default), `"manual"`, `"disabled"`

---

## Code Locations

- **Detection logic:** `src/detector/ide-detector.ts` (new)
- **File generator integration:** `src/generator/file-generator.ts` (modify `buildFileSpecs`)
- **Tests:** `src/detector/ide-detector.test.ts`
- **Type definitions:** Already in `src/types.ts` (add `DetectionResult` interface)

---

## Example Test Cases

```typescript
describe('ide-detector', () => {
  it('detects VS Code via VSCODE_PID', async () => {
    const oldEnv = process.env.VSCODE_PID;
    process.env.VSCODE_PID = '12345';
    const result = await detectIDEs('.');
    expect(result.isVSCode).toBe(true);
    process.env.VSCODE_PID = oldEnv;
  });
  
  it('detects Cursor via .cursor directory', async () => {
    // Create temp workspace with .cursor/
    const result = await detectIDEs(tmpDir);
    expect(result.isCursor).toBe(true);
  });
  
  it('detects Claude Code via .mcp.json', async () => {
    const result = await detectIDEs(tmpDirWithMcpJson);
    expect(result.isClaudeCode).toBe(true);
  });
  
  it('handles multiple detected IDEs', async () => {
    // Workspace with both .cursor and .mcp.json
    const result = await detectIDEs(tmpDirMulti);
    expect(result.detectedIDEs).toContain('cursor');
    expect(result.detectedIDEs).toContain('claude-code');
    expect(result.primaryIDE).toBeNull(); // Ambiguous
  });
});
```

---

## Summary

| Question | Answer |
|---|---|
| Can we detect IDEs? | ✅ Yes, reliably |
| Effort to implement? | Low (100-200 LoC) |
| Breaking changes? | None (opt-in, backward compatible) |
| Performance impact? | Negligible (~1-5ms per scan) |
| Timeline? | Shipped in v1.0.0 (2026-04-17) |
