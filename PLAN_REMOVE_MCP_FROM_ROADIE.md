# Plan: Remove Claude/MCP Code from Roadie Extension

**Status:** Approved  
**Scope:** Remove all MCP/Claude server infrastructure from roadie-App  
**Impact:** Simplify Roadie extension, defer MCP to separate package  
**Timeline:** v0.7.11  

---

## What to Remove

### 1. MCP Config Generation (mcp-config.ts)
**File:** `src/generator/templates/mcp-config.ts`
- Generates `.mcp.json`
- Not used in current Roadie
- Remove entirely

**Action:** Delete file

---

### 2. Claude Hooks Generation (claude-hooks.ts)
**File:** `src/generator/templates/claude-hooks.ts`
- Generates `.claude/settings.json` with hooks
- Not used in current Roadie
- Remove entirely

**Action:** Delete file

---

### 3. File Generator Integration Points
**File:** `src/generator/file-generator.ts`

**Current state:**
```typescript
function buildFileSpecs(learningDb?: LearningDatabase, detection?: DetectionResult): FileSpec[] {
  const specs: FileSpec[] = [ ... base specs ... ];

  // Phase 2: Conditionally add tool-specific files
  if (detection?.isClaudeCode) {
    // Future: add mcp-config and claude-hooks specs here
  }
  if (detection?.isWindsurf) {
    // Future: add windsurf-rules spec here
  }

  return specs;
}
```

**Action:** 
- Remove all Phase 2 comments (mcp-config, claude-hooks, windsurf)
- Keep IDE detection (it's useful for future)
- Simplify to:
```typescript
function buildFileSpecs(learningDb?: LearningDatabase): FileSpec[] {
  return [
    { type: 'copilot_instructions', ... },
    { type: 'agents_md', ... },
    { type: 'claude_md', ... },
    { type: 'cursor_rules', ... },
  ];
}
```

**Keep:** IDE detection (useful, no harm)

---

### 4. IDE Detection Removal (Optional)
**File:** `src/detector/ide-detector.ts`

**Decision:** Keep or remove?
- **Keep:** ✅ Useful for future (IDE-aware logging, preferences)
- **Remove:** ✅ Also fine (Roadie works without it)

**Recommendation:** Keep it. It's harmless and enables future features.

---

### 5. Types Related to MCP
**File:** `src/types.ts`

**Search for and remove:**
- `McpConfig` interface (if exists)
- `McpServerEntry` interface (if exists)
- Any type prefixed with `Mcp`
- Any comments mentioning "Phase 2 MCP"

**Action:** Grep for `Mcp`, remove types

---

### 6. CLAUDE.md Template (Decide)
**File:** `src/generator/templates/claude-md.ts`

**Current purpose:** Generic Claude guidance
**Decision:**
- **Keep:** It's useful (Claude users can reference it)
- **Update:** Remove any MCP/hooks references
- **Clarify:** Make clear it's for manual reference, not auto-integration

**Action:** Update CLAUDE.md template to remove MCP language

---

### 7. Documentation Files
**Files:**
- `roadie_docs/00_CURRENT_STATE.md` — Update v0.7.11 notes
- `roadie_docs/TOOL_INTEGRATION_STATUS.md` — Update (Claude Code section)
- `roadie_docs/IDE_DETECTION_PROPOSAL.md` — Remove or archive
- `roadie_docs/ARCHITECTURE_MISMATCH.md` — Archive (no longer applies)
- `roadie_docs/SEPARATE_MCP_PROPOSAL.md` — Keep but clarify (external)

**Action:**
- Update CURRENT_STATE to say "MCP moved to separate package"
- Update TOOL_INTEGRATION_STATUS to clarify: "Roadie generates context files; separate MCP provides tool access"
- Archive/remove MCP-in-Roadie proposal docs

---

## Files to Delete

1. `src/generator/templates/mcp-config.ts`
2. `src/generator/templates/mcp-config.ts` tests (if exist)
3. `src/generator/templates/claude-hooks.ts`
4. `src/generator/templates/claude-hooks.ts` tests (if exist)
5. `roadie_docs/IDE_DETECTION_PROPOSAL.md` (archive or delete)
6. `roadie_docs/ARCHITECTURE_MISMATCH.md` (archive or delete)

---

## Files to Edit

1. `src/generator/file-generator.ts`
   - Remove Phase 2 MCP/Windsurf comments
   - Simplify buildFileSpecs()
   - Keep IDE detection

2. `src/generator/templates/claude-md.ts`
   - Remove any mention of MCP, hooks, .mcp.json
   - Keep generic Claude guidance
   - Clarify: "This file is for manual reference"

3. `src/types.ts`
   - Remove any Mcp* types
   - Remove Phase 2 comments

4. `roadie_docs/CURRENT_STATE.md`
   - Update v0.7.11 section
   - Note: "MCP integration moved to separate roadie-claude-connector package"
   - Link to separate package docs

5. `roadie_docs/TOOL_INTEGRATION_STATUS.md`
   - Update Claude Code section: "Roadie generates CLAUDE.md; separate MCP provides tool access"
   - Remove integrated MCP references
   - Add link to roadie-claude-connector

6. `README.md` (roadie-App/)
   - Remove any MCP references
   - Clarify: "Roadie generates context files for any tool"
   - Add note: "For Claude Code integration, install roadie-claude-connector"

---

## Testing After Removal

Run:
```bash
npm test          # All tests should pass
npm run build     # Build should succeed
npm run lint      # No errors
```

**Expected result:** All 688 tests pass, no linting errors, clean build

---

## What Remains

✅ **Kept:**
- IDE detection (useful, harmless)
- CLAUDE.md generation (useful context file)
- AGENTS.md generation (useful context file)
- .github/copilot-instructions.md (for Copilot)
- .cursor/rules/ (for Cursor)
- All context file generation

❌ **Removed:**
- MCP config generation (.mcp.json)
- Claude hooks generation
- All Phase 2 MCP infrastructure
- Comments about building MCP in Roadie

---

## Summary

**Before:**
- Roadie: extension + MCP server + context files
- Complexity: High (multiple concerns)
- Scope: Too broad

**After:**
- Roadie: extension + context files only
- Complexity: Lower (single concern)
- Scope: Focused

**Separate MCP (roadie-claude-connector):**
- Takes Roadie's data (SQLite, JSON files)
- Provides MCP tools for Claude Code
- Evolves independently
- Works with or without Roadie extension

---

## Effort Estimate

- Delete 2 template files: 5 min
- Update 5 docs: 10 min
- Update file-generator.ts: 5 min
- Update claude-md.ts: 5 min
- Test & verify: 10 min

**Total: ~35 minutes**

---

## Files Involved

**Delete:**
- `src/generator/templates/mcp-config.ts`
- `src/generator/templates/claude-hooks.ts`

**Edit:**
- `src/generator/file-generator.ts` (remove Phase 2 comments)
- `src/generator/templates/claude-md.ts` (clarify manual reference)
- `src/types.ts` (remove Mcp* types if any)
- `roadie-App/README.md` (update tool integration section)
- `roadie_docs/CURRENT_STATE.md` (v0.7.11 notes)
- `roadie_docs/TOOL_INTEGRATION_STATUS.md` (clarify separate MCP)

**Archive/Delete:**
- `roadie_docs/IDE_DETECTION_PROPOSAL.md`
- `roadie_docs/ARCHITECTURE_MISMATCH.md`
