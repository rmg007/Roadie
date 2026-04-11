# ⚙️ Phase 1.5 Configuration Schema

## All Settings for Phase 1 + Phase 1.5, with Types, Defaults, and Validation

---

## Design Principle

**Every boolean defaults to `false`.** Roadie does nothing the developer hasn't consented to. A fresh install behaves exactly like Phase 1 until the first workflow populates the project model.

---

## Extension Manifest (`package.json` contributes.configuration)

```json
{
  "roadie": {
    "type": "object",
    "title": "Roadie",
    "properties": {
      "roadie.testCommand": {
        "type": "string",
        "default": "",
        "description": "Custom test command override. If empty, Roadie detects from package.json scripts."
      },
      "roadie.modelPreference": {
        "type": "string",
        "enum": ["economy", "balanced", "quality"],
        "default": "balanced",
        "description": "Model selection strategy. 'economy' prefers free models, 'quality' prefers premium."
      },
      "roadie.telemetry": {
        "type": "boolean",
        "default": false,
        "description": "Enable anonymous usage telemetry."
      },
      "roadie.autoCommit": {
        "type": "boolean",
        "default": false,
        "description": "Automatically stage and commit generated .github/ file changes."
      },
      "roadie.editTracking": {
        "type": "boolean",
        "default": false,
        "description": "Track developer modifications to generated files for learning."
      },
      "roadie.workflowHistory": {
        "type": "boolean",
        "default": false,
        "description": "Store workflow execution outcomes in the learning database."
      },
      "roadie.fileWatcherTimeout": {
        "type": "number",
        "default": 500,
        "minimum": 100,
        "maximum": 5000,
        "description": "Debounce time for file watcher events in milliseconds."
      },
      "roadie.maxWatchedPaths": {
        "type": "number",
        "default": 5000,
        "minimum": 100,
        "maximum": 50000,
        "description": "Maximum file paths to watch before switching to polling."
      },
      "roadie.fileWatcherUsePolling": {
        "type": "boolean",
        "default": false,
        "description": "Force polling mode for file watching (slower, more compatible)."
      },
      "roadie.learningDbRetention": {
        "type": "number",
        "default": 90,
        "minimum": 7,
        "maximum": 365,
        "description": "Days to retain learning database entries before pruning."
      },
      "roadie.testTimeout": {
        "type": "number",
        "default": 300,
        "minimum": 30,
        "maximum": 600,
        "description": "Maximum time in seconds for test execution in workflows."
      }
    }
  }
}
```

---

## Settings Reference

### Phase 1 Settings

| Setting | Type | Default | Phase | Used By |
| --- | --- | --- | --- | --- |
| `roadie.testCommand` | string | `""` (auto-detect) | 1 | Workflow Engine (bug fix, refactor) |
| `roadie.modelPreference` | enum | `"balanced"` | 1 | Model Resolver |
| `roadie.telemetry` | boolean | `false` | 1 | Extension Shell |
| `roadie.testTimeout` | number | `300` | 1 | Step Executor |

### Phase 1.5 Settings

| Setting | Type | Default | Phase | Used By |
| --- | --- | --- | --- | --- |
| `roadie.autoCommit` | boolean | `false` | 1.5 | File Generator Manager |
| `roadie.editTracking` | boolean | `false` | 1.5 | Edit Tracker |
| `roadie.workflowHistory` | boolean | `false` | 1.5 | Learning Database |
| `roadie.fileWatcherTimeout` | number | `500` | 1.5 | File Watcher Manager |
| `roadie.maxWatchedPaths` | number | `5000` | 1.5 | File Watcher Manager |
| `roadie.fileWatcherUsePolling` | boolean | `false` | 1.5 | File Watcher Manager |
| `roadie.learningDbRetention` | number | `90` | 1.5 | Learning Database (pruning) |

---

## TypeScript Config Interface

```tsx
interface RoadieConfig {
  // Phase 1
  testCommand: string;
  modelPreference: 'economy' | 'balanced' | 'quality';
  telemetry: boolean;
  testTimeout: number;
  
  // Phase 1.5
  autoCommit: boolean;
  editTracking: boolean;
  workflowHistory: boolean;
  fileWatcherTimeout: number;
  maxWatchedPaths: number;
  fileWatcherUsePolling: boolean;
  learningDbRetention: number;
}

// Load from VS Code settings
function loadConfig(): RoadieConfig {
  const config = vscode.workspace.getConfiguration('roadie');
  return {
    testCommand: config.get('testCommand', ''),
    modelPreference: config.get('modelPreference', 'balanced'),
    telemetry: config.get('telemetry', false),
    testTimeout: config.get('testTimeout', 300),
    autoCommit: config.get('autoCommit', false),
    editTracking: config.get('editTracking', false),
    workflowHistory: config.get('workflowHistory', false),
    fileWatcherTimeout: config.get('fileWatcherTimeout', 500),
    maxWatchedPaths: config.get('maxWatchedPaths', 5000),
    fileWatcherUsePolling: config.get('fileWatcherUsePolling', false),
    learningDbRetention: config.get('learningDbRetention', 90),
  };
}
```

---

## Validation Rules

```tsx
const ConfigSchema = z.object({
  testCommand: z.string().max(500),
  modelPreference: z.enum(['economy', 'balanced', 'quality']),
  telemetry: z.boolean(),
  testTimeout: z.number().min(30).max(600),
  autoCommit: z.boolean(),
  editTracking: z.boolean(),
  workflowHistory: z.boolean(),
  fileWatcherTimeout: z.number().min(100).max(5000),
  maxWatchedPaths: z.number().min(100).max(50000),
  fileWatcherUsePolling: z.boolean(),
  learningDbRetention: z.number().min(7).max(365),
});
```

---

## Standalone Mode Config (Phase 2)

In standalone mode (MCP server without VS Code), settings are read from:

1. `.vscode/settings.json` `roadie.*` keys (if file exists)
2. Environment variables: `ROADIE_TEST_COMMAND`, `ROADIE_MODEL_PREFERENCE`, etc.
3. CLI arguments: `--test-command`, `--model-preference`, etc.

Priority: CLI args > env vars > settings.json > defaults

See Phase 2 Core/Shell Split — `FileConfigProvider` implements this resolution chain.

---

## How Settings Affect Behavior

### Fresh Install (all defaults)

- Extension activates with Phase 1 only
- No file watching, no edit tracking, no workflow history
- Model selection: balanced (standard models preferred)
- No telemetry, no auto-commit
- When first workflow runs: project model created, `.github/` files generated

### Developer enables `editTracking: true`

- Edit Tracker starts monitoring `.github/` files for human modifications
- Snapshots stored in learning database
- Section Manager uses edit history for smarter merge decisions

### Developer enables `workflowHistory: true`

- Workflow outcomes (success/failure, duration, model tiers used) logged to learning database
- `query_workflow_history` MCP tool returns data
- Workflow stats (completion rate, average duration) available

### Developer enables `autoCommit: true`

- After File Generator Manager writes `.github/` files, automatically:
    1. `git add .github/`
    2. `git commit -m "chore: update AI configuration (Roadie)"`
- Only commits if files actually changed (diff-before-write)
- Never commits if there are uncommitted changes in non-.github/ files