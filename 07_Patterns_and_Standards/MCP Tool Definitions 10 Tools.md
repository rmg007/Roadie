# 🔧 MCP Tool Definitions (10 Tools)

## Complete Specifications for All 10 MCP Tools

---

## Tool Summary

| Tool | Category | Read/Write | LLM Required | Standalone |
| --- | --- | --- | --- | --- |
| `roadie/analyze_project` | Project | Write (DB) | No | ✅ Full |
| `roadie/get_project_context` | Project | Read | No | ✅ Full |
| `roadie/rescan_project` | Project | Write (DB) | No | ✅ Full |
| `roadie/run_workflow` | Workflow | Write (files+DB) | Yes | ⚠️ API key |
| `roadie/get_workflow_status` | Workflow | Read | No | ✅ Full |
| `roadie/generate_file` | Generator | Write (files+DB) | No | ✅ Full |
| `roadie/generate_all_files` | Generator | Write (files+DB) | No | ✅ Full |
| `roadie/query_patterns` | Query | Read | No | ✅ Full |
| `roadie/query_workflow_history` | Query | Read | No | ✅ Full |
| `roadie/get_recommendations` | Query | Read | No | ✅ Full |

---

## 1. roadie/analyze_project

**Description:** Scan the project structure and return tech stack, patterns, directory structure, and commands. Triggers a fresh analysis if the model is stale.

**Input Schema:**

```json
{
  "type": "object",
  "properties": {
    "scope": {
      "type": "string",
      "enum": ["full", "dependencies", "patterns", "structure"],
      "default": "full"
    },
    "force": {
      "type": "boolean",
      "default": false
    }
  }
}
```

**Output:**

```json
{
  "techStack": [{"category": "framework", "name": "Next.js", "version": "14.2.0", "sourceFile": "package.json"}],
  "directories": [{"path": "src/", "type": "source", "language": "typescript"}],
  "commands": [{"name": "test", "command": "vitest", "type": "test"}],
  "patterns": [{"category": "export_style", "description": "Named exports only", "confidence": 0.85}],
  "analyzedAt": "2026-04-10T12:00:00Z"
}
```

**Internal module:** `ProjectAnalyzer.analyze(scope)` → `ProjectModel` getters

**Side effects:** Writes analysis results to SQLite. No file system changes outside `.github/.roadie/`.

**Errors:**

- `PROJECT_NOT_FOUND` — project root does not exist
- `ANALYSIS_TIMEOUT` — analysis exceeded 30s

---

## 2. roadie/get_project_context

**Description:** Return the serialized project model as text suitable for LLM prompt injection.

**Input Schema:**

```json
{
  "type": "object",
  "properties": {
    "maxTokens": {"type": "integer", "minimum": 100, "maximum": 50000},
    "scope": {"type": "string", "enum": ["full", "stack", "structure", "commands", "patterns"], "default": "full"},
    "relevantPaths": {"type": "array", "items": {"type": "string"}}
  }
}
```

**Output:**

```json
{
  "context": "## Tech Stack\n- TypeScript 5.4\n- Next.js 14.2...",
  "tokenEstimate": 1250,
  "truncated": false
}
```

**Internal module:** `ProjectModel.toContext(options)`

**Side effects:** None (read-only).

**Errors:**

- `MODEL_EMPTY` — returns empty context with warning to run `analyze_project` first

---

## 3. roadie/rescan_project

**Description:** Force a full re-scan of the project. Replaces file watcher in standalone mode.

**Input:** Empty object `{}`

**Output:**

```json
{
  "status": "completed",
  "durationMs": 2500,
  "techStackEntries": 12,
  "directoriesScanned": 45,
  "patternsDetected": 6
}
```

**Internal module:** `ProjectAnalyzer.analyze('full')` with forced invalidation.

---

## 4. roadie/run_workflow

**Description:** Trigger a named workflow. Returns the complete result when finished. Long-running call.

**Input Schema:**

```json
{
  "type": "object",
  "required": ["workflow", "prompt"],
  "properties": {
    "workflow": {"type": "string", "enum": ["bug_fix", "feature", "refactor", "review", "document", "dependency", "onboard"]},
    "prompt": {"type": "string"},
    "options": {
      "type": "object",
      "properties": {
        "modelPreference": {"type": "string", "enum": ["economy", "balanced", "quality"], "default": "balanced"},
        "testTimeout": {"type": "integer", "default": 300},
        "testCommand": {"type": "string"},
        "autoApprove": {"type": "boolean", "default": true}
      }
    }
  }
}
```

**Output:**

```json
{
  "executionId": "wf_abc123",
  "workflow": "bug_fix",
  "status": "completed",
  "stepsCompleted": 8,
  "stepsTotal": 8,
  "durationMs": 45000,
  "result": {
    "summary": "Fixed null reference in profile serializer...",
    "filesModified": ["src/serializers/profile.ts"],
    "testsRun": true,
    "testsPassed": true
  }
}
```

**Internal module:** `WorkflowEngine.execute()` (bypasses IntentClassifier — workflow specified directly).

**Side effects:** Modifies source files, creates test files, runs shell commands. **Most impactful tool.**

**Standalone behavior:**

- `NullModelProvider` active → returns `LLM_UNAVAILABLE` error
- `DirectAPIModelProvider` configured → workflows execute via direct API
- Feature workflow: `autoApprove` defaults to `true` in standalone (no interactive UI)

**Errors:**

- `LLM_UNAVAILABLE` — no model provider configured
- `WORKFLOW_NOT_FOUND` — unknown workflow type
- `WORKFLOW_FAILED` — failed after all retries (includes `errorSummary`)

---

## 5. roadie/get_workflow_status

**Description:** Check progress of a running workflow.

**Input:** `{"executionId": "wf_abc123"}`

**Output:**

```json
{
  "executionId": "wf_abc123",
  "workflow": "bug_fix",
  "status": "running",
  "currentStep": "verify_fix",
  "stepsCompleted": 3,
  "stepsTotal": 8,
  "elapsedMs": 15000
}
```

**Internal module:** `WorkflowEngine.getActiveWorkflows()`

**Errors:** `EXECUTION_NOT_FOUND` — unknown execution ID

---

## 6. roadie/generate_file

**Description:** Generate or regenerate a specific `.github/` file.

**Input Schema:**

```json
{
  "type": "object",
  "required": ["fileType"],
  "properties": {
    "fileType": {
      "type": "string",
      "enum": ["copilot-instructions", "agents-md", "typescript-instructions", "react-instructions", "python-instructions", "debugger-agent", "reviewer-agent", "hooks", "pr-template", "issue-templates", "mcp-config"]
    },
    "force": {"type": "boolean", "default": false}
  }
}
```

**Output:**

```json
{
  "filePath": ".github/copilot-instructions.md",
  "status": "updated",
  "humanEditsPreserved": true,
  "contentHash": "sha256:abc123..."
}
```

**Status values:** `created` | `updated` | `unchanged` | `skipped` (markers removed)

**Internal module:** `FileGenerator.generate(fileType)`

**Side effects:** Writes/modifies `.github/` files. Respects section ownership.

---

## 7. roadie/generate_all_files

**Description:** Regenerate all `.github/` files from current project model.

**Input:** `{"force": false}`

**Output:**

```json
{
  "files": [
    {"filePath": ".github/copilot-instructions.md", "status": "updated"},
    {"filePath": "AGENTS.md", "status": "unchanged"}
  ],
  "totalGenerated": 8,
  "totalUpdated": 3,
  "totalUnchanged": 5,
  "durationMs": 1200
}
```

**Internal module:** `FileGenerator.generateAll()`

---

## 8. roadie/query_patterns

**Description:** Return discovered coding patterns.

**Input Schema:**

```json
{
  "type": "object",
  "properties": {
    "category": {"type": "string", "enum": ["export_style", "test_convention", "error_handling", "import_ordering", "commit_convention", "async_patterns", "all"], "default": "all"},
    "minConfidence": {"type": "number", "minimum": 0, "maximum": 1, "default": 0.5}
  }
}
```

**Output:**

```json
{
  "patterns": [
    {
      "category": "export_style",
      "description": "Named exports only (no default exports in 45 files)",
      "confidence": 0.92,
      "evidence": ["src/utils/helpers.ts:1"],
      "detectedAt": "2026-04-10T12:00:00Z"
    }
  ]
}
```

**Internal module:** `ProjectModel.getPatterns()` filtered by params.

---

## 9. roadie/query_workflow_history

**Description:** Return past workflow outcomes. Requires `roadie.workflowHistory` enabled.

**Input:**

```json
{
  "limit": 20,
  "workflowType": "bug_fix",
  "status": "completed"
}
```

**Output:**

```json
{
  "entries": [{"workflowType": "bug_fix", "status": "completed", "durationMs": 45000, "createdAt": "..."}],
  "totalCount": 42,
  "historyEnabled": true
}
```

**When disabled:** Returns `{"entries": [], "historyEnabled": false, "message": "Enable roadie.workflowHistory in settings."}`

---

## 10. roadie/get_recommendations

**Description:** Actionable recommendations for improving AI configuration.

**Input:** `{}`

**Output:**

```json
{
  "recommendations": [
    {
      "priority": "high",
      "category": "missing_config",
      "title": "No copilot-instructions.md found",
      "description": "Run roadie/generate_file to create project-aware Copilot instructions.",
      "action": "roadie/generate_file",
      "actionArgs": {"fileType": "copilot-instructions"}
    }
  ]
}
```

**Recommendation categories:** `missing_config` | `stale_model` | `incomplete_patterns` | `unused_features` | `configuration_suggestion`

---

## Error Response Format

All tools use a consistent error format:

```json
{
  "error": "Human-readable error message",
  "code": "ERROR_CODE",
  "details": {}
}
```

**Error codes:** `PROJECT_NOT_FOUND` | `MODEL_EMPTY` | `MODEL_STALE` | `LLM_UNAVAILABLE` | `WORKFLOW_NOT_FOUND` | `WORKFLOW_FAILED` | `EXECUTION_NOT_FOUND` | `FILE_PERMISSION_ERROR` | `DATABASE_ERROR` | `INVALID_INPUT` | `HISTORY_DISABLED` | `ANALYSIS_TIMEOUT`

## Input Validation

All tool inputs are validated with Zod schemas before execution. Invalid inputs return `INVALID_INPUT` with Zod error details.

```tsx
const AnalyzeProjectInput = z.object({
  scope: z.enum(['full', 'dependencies', 'patterns', 'structure']).default('full'),
  force: z.boolean().default(false),
});
```