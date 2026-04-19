# Roadie — File Generation Strategy

**Last Updated:** 2026-04-15

---

## Current Approach (v1.0.0): Static Generation with IDE Awareness

Roadie generates a **fixed set of files** unconditionally. IDE detection (`src/detector/ide-detector.ts`) is available for logging and future conditional generation, but v1.0.0 still generates all files regardless of detected IDEs.

### Files Always Generated

| File | Purpose | Format | Primary Consumer |
|---|---|---|---|
| `.github/copilot-instructions.md` | Copilot context injection | Markdown (Copilot-specific keywords) | GitHub Copilot |
| `AGENTS.md` | Generic agent guidance | Markdown (generic) | Any AI agent (Copilot, Claude, Gemini, Cursor, etc.) |
| `CLAUDE.md` | Claude-specific workspace guidance | Markdown (generic) | Claude Code, Claude web/API (when manually referenced) |
| `.cursor/rules/project.mdc` | Project-level Cursor rules | Cursor markdown | Cursor IDE |
| `.cursor/rules/*.mdc` (per-dir) | Directory-scoped Cursor rules | Cursor markdown | Cursor IDE |
| `.github/instructions/*` | Path-scoped context | Markdown (generic) | Any tool reading `.github/` |
| `.roadie/last-scan.json` | Project metadata | JSON | Roadie internal, MCP tools (Phase 2) |

### Why No Detection?

1. **Roadie is a VS Code extension** — it doesn't know what other tools are installed or being used
2. **No telemetry to detect tools** — Roadie doesn't ship with tool detection (intentional privacy choice)
3. **Generating everything is safe** — Extra files don't hurt; tools simply ignore files they don't recognize
4. **Cost of detection exceeds benefit** — Checking for installed tools, IDE capabilities, etc. adds complexity for minimal gain

---

## What This Means

### You Get All Files Regardless

Even if you:
- Only use GitHub Copilot → you still get `.cursor/rules/`, `CLAUDE.md`, `AGENTS.md` (harmless extras)
- Only use Cursor → you still get `copilot-instructions.md`, `CLAUDE.md`, `AGENTS.md` (harmless extras)
- Only use Claude Code → you still get all of the above (but no `.mcp.json` or hooks, those are Phase 2)
- Don't have any AI tool → you still get all files (they're just there if you add a tool later)

### No Environment Variables or Config

Roadie generation has zero settings for:
- `"roadie.generateForTools"` — doesn't exist
- `"roadie.enableCursor"` or `"roadie.enableClaude"` — doesn't exist
- IDE detection in `.vscode/settings.json` — not consulted

### Phase 2 Will Change This

Phase 2 is planned to add **tool-specific conditional generation**:
- Detect if `.mcp.json` exists → if so, generate `.mcp.json` with Roadie server entry
- Detect if `~/.claude/settings.json` exists → if so, generate hooks for Claude Code
- (Future) Detect Windsurf and generate Windsurf-specific config
- (Future) VS Code extension coordination

But for v1.0.0, it's **all-or-nothing static generation** (with IDE detection available but not yet used to branch generation).

---

## Implementation Detail

### In Code

```typescript
// src/generator/file-generator.ts
function buildFileSpecs(learningDb?: LearningDatabase): FileSpec[] {
  return [
    { type: 'copilot_instructions', path: COPILOT_INSTRUCTIONS_PATH, ... },
    { type: 'agents_md', path: AGENTS_MD_PATH, ... },
    { type: 'claude_md', path: CLAUDE_MD_PATH, ... },
    { type: 'cursor_rules', path: CURSOR_RULES_PATH, ... },
  ];
}

async generateAll(model: ProjectModel): Promise<GeneratedFile[]> {
  for (const spec of this.fileSpecs) {  // Always iterates same specs
    const result = await this.generateFile(spec, model);
    results.push(result);
  }
}
```

No branching. No detection. Same files every time.

### Future Phase 2 Will Look Like

```typescript
// Conceptual (not implemented)
function buildFileSpecs(...): FileSpec[] {
  const specs = [ ... base files ... ];
  
  // Phase 2: Conditional generation
  if (fileSystem.exists('.mcp.json')) {
    specs.push({ type: 'mcp_config', ... });
  }
  if (fileSystem.exists('~/.claude/settings.json')) {
    specs.push({ type: 'claude_hooks', ... });
  }
  if (fileSystem.exists('.windsurf')) {
    specs.push({ type: 'windsurf_rules', ... });
  }
  
  return specs;
}
```

But this is NOT in v1.0.0 (planned for v1.1+).

---

## Artifacts Exist But Unused (Phase 2 Prep)

| Template File | Status | When It'll Be Used |
|---|---|---|
| `src/generator/templates/mcp-config.ts` | ✅ Exists | Phase 2 (when MCP server is built) |
| `src/generator/templates/claude-hooks.ts` | ✅ Exists | Phase 2 (when Claude hooks are implemented) |
| `src/generator/templates/windsurf-rules.ts` | ❌ Missing | Phase 2 (design pending) |

These templates are **forward-designed** but **not called** from `buildFileSpecs()`.

---

## Implications for Users

### ✅ Good News
- **No configuration needed** — just install Roadie, it works everywhere
- **Future-proof** — if you add a new tool later, its config files are already there
- **Portable** — generated files travel with the code; they help any tool that touches the repo

### ⚠️ Limitation
- **No optimization for YOUR setup** — you get a one-size-fits-all solution
- **Extra files** — if you only use Copilot, you're getting Cursor rules and Claude instructions (harmless but unnecessary)
- **Tool-specific features unused** — Claude Code can't use hooks yet (Phase 2), Windsurf has no native integration (Phase 2)

---

## Recommendation for Phase 2

When Phase 2 ships, consider adding:

1. **Opt-in tool detection** (with opt-out):
   ```json
   {
     "roadie.generateFor": ["copilot", "cursor", "claude"],
     "roadie.autoDetectTools": true
   }
   ```

2. **Tool-specific config merging** — `.mcp.json` → detect if it exists, add Roadie entry if needed

3. **IDE-aware context** — If running in Claude Code, generate hooks; if in Cursor, optimize rule generation

4. **Notification system** — Alert user if they add a new tool that Roadie can integrate with

---

## Summary

| Question | Answer |
|---|---|
| Does Roadie detect IDEs? | ✅ Yes (v1.0.0 `detector` module) |
| Does Roadie generate different files for different tools? | ❌ No (static set; conditional generation is Phase 2) |
| Can I configure what to generate? | ❌ No settings (Phase 2 planned) |
| Will Phase 2 add conditional generation? | ✅ Yes (planned for v1.1+) |
| Are the extra files harmful? | ❌ No, tools ignore what they don't recognize |
