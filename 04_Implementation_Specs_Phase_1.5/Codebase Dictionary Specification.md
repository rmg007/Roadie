# Build Prompt: Codebase Dictionary (Phase 1.5 Addition)
You are building the Codebase Dictionary feature for the Roadie VS Code extension.
This is an additive Phase 1.5 module. Phase 1 and Phase 1.5 must continue
working unchanged after this build. Read 00_START_HERE.md before proceeding.

---

## WHAT YOU ARE BUILDING

A system that lets Roadie's workflows annotate every code entity they
generate or modify, stores those annotations in SQLite, and surfaces them
as rich context in future workflow prompts. The result is a living,
queryable knowledge graph of the codebase — built incrementally as
Roadie works, not by scanning.

---

## MODULE 1: src/dictionary/entity-writer.ts

### Purpose
Called by AgentSpawner after every successful code-writing step.
Parses the agent's output to extract entities (functions, classes,
interfaces, routes, models, enums, constants) and writes them to SQLite.

### Public Interface

```ts
/**
 * @module entity-writer
 * @description Extracts code entities from agent output and persists
 *   them to the codebase_entities and entity_relationships tables.
 * @depends-on model/database.ts (SQLite connection)
 * @depended-on-by spawner/agent-spawner.ts
 */

export interface EntityWriter {
  /**
   * Extract entities from agent output and upsert into SQLite.
   * Called after every step of kind 'implementation' or 'refactor'.
   */
  recordEntities(params: RecordEntitiesParams): Promise<void>;

  /**
   * Mark all entities in a file as stale when the file is deleted
   * or the File Watcher fires a DELETE event for it.
   */
  invalidateFile(filePath: string): Promise<void>;
}

export interface RecordEntitiesParams {
  /** Absolute path to the file that was written */
  filePath: string;
  /** Full content of the file after the agent wrote it */
  fileContent: string;
  /** Which workflow produced this code */
  workflowType: string;
  /** Which step within the workflow produced this code */
  stepId: string;
  /** The developer's original prompt that started the workflow */
  originalPrompt: string;
}

export interface CodeEntity {
  name: string;
  kind: 'function' | 'class' | 'interface' | 'type' | 'enum'
      | 'constant' | 'route' | 'model' | 'component';
  filePath: string;
  lineNumber: number;
  signature: string;
  /** One-sentence purpose extracted from JSDoc or inferred from name+body */
  purpose: string;
  isExported: boolean;
  createdByWorkflow: string;
  createdAt: string;  // ISO 8601
}

export interface EntityRelationship {
  sourceEntityName: string;
  sourceFilePath: string;
  targetEntityName: string;
  targetFilePath: string;
  /** 'calls' | 'imports' | 'extends' | 'implements' | 'uses' | 'returns' */
  relationship: string;
}
```

### Extraction Rules (regex-based, NOT TypeScript compiler API)

Extract the following from file content using regex:

| Pattern | Kind | Regex hint |
|---|---|---|
| `export function X` / `export async function X` | function | `/export\s+(async\s+)?function\s+(\w+)/g` |
| `export const X = (...) =>` | function | `/export\s+const\s+(\w+)\s*=\s*(\(.*?\)|async\s*\()/g` |
| `export class X` | class | `/export\s+(abstract\s+)?class\s+(\w+)/g` |
| `export interface X` | interface | `/export\s+interface\s+(\w+)/g` |
| `export type X` | type | `/export\s+type\s+(\w+)/g` |
| `export enum X` | enum | `/export\s+enum\s+(\w+)/g` |
| `export const X =` (non-function) | constant | match remaining exports |
| `app.get('/path'`, `router.post('/path'` | route | `/(?:app\|router)\.(get\|post\|put\|patch\|delete)\s*\(\s*['"\`]([^'"\`]+)/g` |
| Prisma model blocks | model | `/model\s+(\w+)\s*\{/g` |

For `signature`: capture the full first line of the declaration (up to `{` or `=>`).
For `purpose`: check the 3 lines above the declaration for a JSDoc `/** ... */` comment.
If no JSDoc exists, set purpose to `""` (empty). Do NOT infer or hallucinate purpose.

For `lineNumber`: count newlines before the match position.

### Relationship Extraction

After extracting entities, scan import statements in the file:
/import\s+{([^}]+)}\s+from\s+'"['"]/g

For each imported name that matches a known entity in the database,
write an `imports` relationship from the current file's entities to
the imported entity. Do not create relationships for names not in the DB.

### Error Handling

- If the file cannot be parsed (binary, minified, >500KB): log warning,
  return early without writing. Never throw.
- If SQLite write fails: log error with entity name and file path.
  Do not propagate — dictionary writes must never crash a workflow.
- If the same entity (file_path + name + kind) already exists: UPSERT,
  update signature/purpose/line_number, append a modification record.

### Tests Required

```ts
describe('EntityWriter', () => {
  it('extracts exported functions from TypeScript file')
  it('extracts exported classes with correct line numbers')
  it('extracts exported interfaces')
  it('extracts Express route definitions')
  it('does not extract non-exported functions')
  it('handles files with no exports gracefully')
  it('handles files >500KB without throwing')
  it('upserts on duplicate file_path+name+kind')
  it('writes modification record when entity already exists')
  it('writes import relationships for known entities')
  it('does not crash workflow if SQLite write fails')
})
```

---

## MODULE 2: src/dictionary/dictionary-query.ts

### Purpose
Provides typed query methods over the codebase_entities and
entity_relationships tables. Used by PromptBuilder to inject
relevant dictionary context into workflow prompts.

### Public Interface

```ts
/**
 * @module dictionary-query
 * @description Typed query API for the codebase dictionary.
 * @depends-on model/database.ts
 * @depended-on-by spawner/prompt-builder.ts, model/project-model.ts
 */

export interface DictionaryQuery {
  /**
   * Get all entities in a set of files. Used by PromptBuilder
   * when the workflow is focused on specific paths.
   */
  getEntitiesInFiles(filePaths: string[]): Promise<CodeEntity[]>;

  /**
   * Get all entities that depend on (import or call) a given entity.
   * Used before refactoring to assess blast radius.
   */
  getDependents(entityName: string, filePath: string): Promise<CodeEntity[]>;

  /**
   * Get all entities that a given entity depends on.
   * Used to understand an entity's requirements before modifying it.
   */
  getDependencies(entityName: string, filePath: string): Promise<CodeEntity[]>;

  /**
   * Full-text search across entity names and purposes.
   * Used by the bug-fix and feature workflows to find relevant code.
   */
  search(query: string, limit?: number): Promise<CodeEntity[]>;

  /**
   * Get a serialized summary of the dictionary suitable for
   * injecting into an LLM prompt. Respects token budget.
   */
  toContext(options?: DictionaryContextOptions): Promise<DictionaryContext>;

  /** Total entity count — used to decide whether to include dictionary in prompts */
  getEntityCount(): Promise<number>;
}

export interface DictionaryContextOptions {
  /** Only include entities from these paths */
  relevantPaths?: string[];
  /** Max characters in the output string (default: 3000) */
  maxChars?: number;
  /** Entity kinds to include (default: all) */
  includeKinds?: CodeEntity['kind'][];
}

export interface DictionaryContext {
  /** Markdown-formatted summary ready for prompt injection */
  summary: string;
  /** Number of entities included */
  entityCount: number;
  /** Whether the output was truncated due to maxChars */
  truncated: boolean;
}
```

### toContext() Output Format

```markdown
## Codebase Dictionary (relevant entities)

### src/auth/login.ts
- `validateToken(token: string): boolean` — function — Validates JWT expiration
- `LoginHandler` — class — Handles POST /auth/login requests
- `AuthError` — interface — Typed error shape for auth failures

### src/services/user.ts
- `UserService` — class — CRUD operations for User model
- `getUserById(id: string): Promise<User>` — function — Fetch user by primary key
```

If entity count is 0: return `summary: ""` (empty string, do not inject).
If truncated: append `\n_[dictionary truncated — showing {n} of {total} entities]_`.

### Tests Required

```ts
describe('DictionaryQuery', () => {
  it('returns empty array for files with no entities')
  it('getEntitiesInFiles returns only entities in specified files')
  it('getDependents returns entities that import the target')
  it('getDependencies returns entities the target imports')
  it('search matches entity names case-insensitively')
  it('search matches entity purposes')
  it('toContext respects maxChars and sets truncated=true')
  it('toContext returns empty string when no entities exist')
  it('toContext groups entities by file path')
})
```

---

## MODULE 3: src/dictionary/dictionary-generator.ts

### Purpose
A FileTypeGenerator (implementing the same interface as the other 8
generators in Phase 1.5) that reads all dictionary entries from SQLite
and produces `.github/codebase-dictionary.md` — a human-readable,
git-committed reference document. Registered in FileGeneratorManager.

### Interface

Implements `FileTypeGenerator` from `04_Implementation_Specs_Phase_1.5/
File Generator Manager Specification.md`. No new public interface.

### Output Format

```markdown
# Codebase Dictionary

> Auto-generated by Roadie. Do not edit directly.
> Last updated: {ISO timestamp}
> Total entities: {count}

## Functions

| Name | File | Signature | Purpose | Created By |
|------|------|-----------|---------|------------|
| `validateToken` | src/auth/login.ts:42 | `(token: string): boolean` | Validates JWT expiration | bug_fix |

## Classes

| Name | File | Purpose | Created By |
|------|------|---------|------------|

## Interfaces & Types

(same table pattern)

## API Routes

| Method | Path | Handler File | Created By |
|--------|------|-------------|------------|

## Database Models

| Name | File | Created By |
|------|------|------------|
```

### Trigger
`triggers: ['techStack', 'workflow_complete']`
Regenerates after any workflow that writes code. Budget: **300ms**.

### Tests Required

```ts
describe('DictionaryGenerator', () => {
  it('produces valid markdown with correct table headers')
  it('groups entities by kind')
  it('handles empty dictionary gracefully (produces header only)')
  it('completes within 300ms for 500 entities')
  it('uses Roadie section markers')
})
```

---

## DATABASE SCHEMA (add to Migration 002)

```sql
-- Add to existing Migration 002 in src/infrastructure/database.ts

CREATE TABLE IF NOT EXISTS codebase_entities (
  id                  INTEGER PRIMARY KEY AUTOINCREMENT,
  name                TEXT    NOT NULL,
  kind                TEXT    NOT NULL CHECK(kind IN (
                        'function','class','interface','type','enum',
                        'constant','route','model','component')),
  file_path           TEXT    NOT NULL,
  line_number         INTEGER,
  signature           TEXT,
  purpose             TEXT    DEFAULT '',
  is_exported         INTEGER NOT NULL DEFAULT 1,
  created_by_workflow TEXT,
  created_at          TEXT    NOT NULL DEFAULT (datetime('now')),
  updated_at          TEXT    NOT NULL DEFAULT (datetime('now')),
  UNIQUE(file_path, name, kind)
);

CREATE INDEX IF NOT EXISTS idx_entities_file
  ON codebase_entities(file_path);
CREATE INDEX IF NOT EXISTS idx_entities_name
  ON codebase_entities(name);

CREATE TABLE IF NOT EXISTS entity_relationships (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  source_id    INTEGER NOT NULL REFERENCES codebase_entities(id) ON DELETE CASCADE,
  target_id    INTEGER NOT NULL REFERENCES codebase_entities(id) ON DELETE CASCADE,
  relationship TEXT    NOT NULL,
  UNIQUE(source_id, target_id, relationship)
);

CREATE TABLE IF NOT EXISTS entity_modifications (
  id                  INTEGER PRIMARY KEY AUTOINCREMENT,
  entity_id           INTEGER NOT NULL REFERENCES codebase_entities(id) ON DELETE CASCADE,
  workflow_type       TEXT,
  step_id             TEXT,
  original_prompt     TEXT,
  change_description  TEXT,
  modified_at         TEXT    NOT NULL DEFAULT (datetime('now'))
);
```

---

## INTEGRATION POINTS (modify existing modules)

### 1. src/spawner/agent-spawner.ts
After a step with `toolScope === 'implementation'` succeeds and
`AgentResult.status === 'success'`, call:

```ts
await entityWriter.recordEntities({
  filePath: /* extracted from tool call results */,
  fileContent: /* read the file after writing */,
  workflowType: context.intent.intent,
  stepId: step.id,
  originalPrompt: context.prompt,
});
```

Extract `filePath` from `AgentResult.toolResults` — look for a tool call
named `edit_file` or `write_file` and read the `input.path` argument.

Do NOT block the step result on this write. Use `Promise.allSettled`
or fire-and-forget with error logging. The dictionary write must
never delay or fail a workflow step.

### 2. src/spawner/prompt-builder.ts
In `build()`, after injecting project model context, call:

```ts
const dictContext = await dictionaryQuery.toContext({
  relevantPaths: extractPathsFromPrompt(config.context),
  maxChars: 3000,
});

if (dictContext.summary) {
  layers.push(dictContext.summary);
}
```

`extractPathsFromPrompt` is a local helper that scans the prompt
and context for file paths (regex: `/\b\S+\.(ts|js|tsx|jsx|py|go|rs)\b/g`).

### 3. src/watcher/file-watcher-manager.ts
When a DELETE event fires for a `.ts`, `.js`, `.tsx`, `.jsx`, `.py` file:

```ts
await entityWriter.invalidateFile(change.filePath);
```

---

## PRUNING

Add to `LearningDatabase.prune()` (src/learning/learning-database.ts):

```ts
// Remove entity_modifications older than 90 days
db.prepare(`
  DELETE FROM entity_modifications
  WHERE modified_at < datetime('now', '-90 days')
`).run();
```

Do NOT prune `codebase_entities` or `entity_relationships` on age —
these are structural facts about the codebase, not time-series data.

---

## FILES TO CREATE
src/dictionary/entity-writer.ts          (~200 lines)
src/dictionary/entity-writer.test.ts     (~250 lines)
src/dictionary/dictionary-query.ts       (~150 lines)
src/dictionary/dictionary-query.test.ts  (~200 lines)
src/dictionary/dictionary-generator.ts  (~120 lines)
src/dictionary/dictionary-generator.test.ts (~80 lines)

## FILES TO MODIFY
src/model/database.ts             — add Migration 002 SQL above
src/spawner/agent-spawner.ts      — call entityWriter after implementation steps
src/spawner/prompt-builder.ts     — inject dictionary context
src/watcher/file-watcher-manager.ts — call invalidateFile on DELETE
src/learning/learning-database.ts — add entity_modifications pruning
src/generator/file-generator-manager.ts — register DictionaryGenerator

---

## DEFINITION OF DONE

- [ ] `npm run test` passes — all new and existing tests green
- [ ] `npm run lint` passes — no ESLint errors
- [ ] `npm run build` exits 0
- [ ] Dictionary entries appear in SQLite after running the bug-fix workflow
      against a real Node.js project (manual verification)
- [ ] `.github/codebase-dictionary.md` is generated with correct content
- [ ] Prompt injection is visible in mock call logs (entity context appears)
- [ ] No workflow step takes longer due to dictionary writes
      (verified by comparing step timing before/after)
- [ ] All 6 test files have ≥ the test cases specified above
- [ ] No file exceeds 300 lines
