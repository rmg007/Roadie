# Phase 2.5 - Adaptive Learning and Self-Improvement

> **WARNING / STATUS:** DESIGN PHASE
> **DO NOT BUILD THIS BEFORE 2028.**
> This phase must not be implemented until Phases 1-2 have been thoroughly shipped, tested, and aggressively used on real projects for an extended period. Developing before substantial usage data is available will lead to untested learning algorithms and degraded performance.

**Prerequisite:** Real usage data from Phases 1-2 (edit tracking history, workflow outcomes, classification corrections)
**Target:** Transform Roadie from a static generator into a system that compounds in quality over time
**Estimated Build Time:** 20-30 hours
**Milestones:** M29-M33

---

## Why Phase 2.5 Exists

After Phase 2, Roadie generates configuration based on project analysis and executes workflows with static prompt templates. It's smart, but it doesn't learn. Two projects with identical tech stacks get identical generated files, even if the developers have completely different preferences.

Phase 2.5 closes the feedback loop: Roadie observes whether its generated configuration actually helps, and adjusts.

---

## What Phase 2.5 Adds

### 1. Generation Quality Scoring

Across all edit tracking data, compute quality scores for each generated file type and section.

```typescript
interface GenerationQualityScore {
  fileType: GeneratedFileType;
  sectionId: string;
  keepRate: number;        // % of times section was kept unchanged (0.0-1.0)
  editRate: number;        // % of times section was modified by developer
  deleteRate: number;      // % of times section was removed entirely
  sampleSize: number;      // number of generation cycles observed
  trend: 'improving' | 'stable' | 'degrading';
  lastUpdated: string;
}
```

**How it works:**
- After each file generation, the Edit Tracker records whether each Roadie-owned section was kept, modified, or deleted
- Quality scores aggregate this data over time
- Sections with >40% delete rate across 10+ cycles are flagged for removal from the template
- Sections with >60% edit rate are flagged for prompt improvement
- Sections with >80% keep rate are validated as high quality

**SQLite schema addition:**
```sql
CREATE TABLE IF NOT EXISTS generation_quality (
  file_type TEXT NOT NULL,
  section_id TEXT NOT NULL,
  keep_count INTEGER NOT NULL DEFAULT 0,
  edit_count INTEGER NOT NULL DEFAULT 0,
  delete_count INTEGER NOT NULL DEFAULT 0,
  last_updated TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (file_type, section_id)
);
```

**Integration:** The `get_recommendations` MCP tool uses quality scores to suggest "Stop generating the commit-conventions section - developers delete it 65% of the time."

---

### 2. Adaptive Prompt Templates

Workflow prompt templates are currently static strings. Phase 2.5 makes them dynamic - the learning database feeds project-specific knowledge into prompts.

**Current (static):**
```
You are a fixer. Generate a fix for this bug.
Project uses: {tech_stack}
```

**Phase 2.5 (adaptive):**
```
You are a fixer. Generate a fix for this bug.
Project uses: {tech_stack}

Learned context from this project:
- Previous bug fixes in auth module required null-coalesce patterns (3 occurrences)
- Developer preference: minimal fixes over rewrites (edit history shows 80% of large refactors were reverted)
- Common failure pattern: test runner needs --forceExit flag (workflow step 4 failed 4 times without it)
{learned_context}
```

**Implementation:**

```typescript
interface AdaptivePromptBuilder extends PromptBuilder {
  /** Inject learned context into the task layer of the prompt */
  injectLearnedContext(
    role: AgentRole,
    projectPath: string,
    workflowType: string
  ): Promise<string>;
}
```

The `injectLearnedContext` method queries:
1. **Workflow outcome patterns** - which steps fail most often for this workflow type, and what fixed them
2. **Developer edit patterns** - what the developer consistently changes in generated output (their preferences)
3. **Codebase dictionary patterns** - which entities were modified most recently and why
4. **Classification corrections** - when the LLM overrode the local classifier, what was the correction pattern

Token budget: learned context is capped at 500 tokens (configurable via `roadie.adaptiveContextBudget`).

---

### 3. Intent Classifier Weight Tuning

The pattern weights in `intent-patterns.ts` are currently hardcoded exact floats. Phase 2.5 adjusts them based on how often the LLM fallback corrects the local classifier.

**The correction signal already exists:** When `requiresLLM` is true, the ChatParticipantHandler prepends the classification prefix, and `parseClassification()` extracts the LLM's intent. If the LLM's intent differs from the local classifier's top intent, that's a correction.

```typescript
interface ClassifierTuningRecord {
  prompt: string;               // the developer's prompt
  localIntent: IntentType;      // what the local classifier picked
  localConfidence: number;      // confidence score
  llmIntent: IntentType;        // what the LLM corrected to
  wasCorrection: boolean;       // localIntent !== llmIntent
  timestamp: string;
}
```

**Tuning algorithm:**
1. Collect correction records over N workflow executions (minimum 50 before tuning)
2. For each pattern that contributed to a misclassification, reduce its weight by 0.05
3. For each pattern that contributed to a correct classification, increase its weight by 0.02 (slower increase to prevent overfitting)
4. Weights are clamped to [0.10, 0.50] range - never zero (pattern still exists), never dominant
5. Tuned weights are stored in SQLite, not in code - the code provides defaults, the database provides overrides
6. A `roadie.resetClassifier` command restores factory weights

**SQLite schema addition:**
```sql
CREATE TABLE IF NOT EXISTS classifier_weight_overrides (
  intent TEXT NOT NULL,
  pattern_index INTEGER NOT NULL,
  original_weight REAL NOT NULL,
  tuned_weight REAL NOT NULL,
  correction_count INTEGER NOT NULL DEFAULT 0,
  last_tuned TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (intent, pattern_index)
);

CREATE TABLE IF NOT EXISTS classifier_corrections (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  prompt_hash TEXT NOT NULL,     -- SHA-256 of prompt (not the prompt itself - privacy)
  local_intent TEXT NOT NULL,
  local_confidence REAL NOT NULL,
  llm_intent TEXT NOT NULL,
  was_correction INTEGER NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

### 4. Workflow Step Optimization

Track which workflow steps consistently fail, which model tiers they escalate to, and what error patterns recur.

```typescript
interface StepPerformanceRecord {
  workflowType: string;
  stepId: string;
  successRate: number;          // 0.0-1.0 across all executions
  averageAttempts: number;      // mean attempts before success
  commonEscalationTier: ModelTier;  // which tier usually succeeds
  commonErrorPattern: string;   // most frequent error substring
  suggestedOptimization: string | null;
}
```

**Automatic optimizations (no human approval needed):**
- If a step fails at Tier 0 more than 70% of the time across 20+ executions, automatically start it at Tier 1
- If a step consistently needs the `--forceExit` flag or similar, add it to the test command automatically
- If a step's prompt template produces parsing errors >30% of the time, log a recommendation to rewrite the template

**SQLite schema addition:**
```sql
CREATE TABLE IF NOT EXISTS step_performance (
  workflow_type TEXT NOT NULL,
  step_id TEXT NOT NULL,
  total_executions INTEGER NOT NULL DEFAULT 0,
  success_count INTEGER NOT NULL DEFAULT 0,
  total_attempts INTEGER NOT NULL DEFAULT 0,
  tier0_failures INTEGER NOT NULL DEFAULT 0,
  tier1_failures INTEGER NOT NULL DEFAULT 0,
  tier2_failures INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  last_updated TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (workflow_type, step_id)
);
```

---

### 5. Developer Preference Learning

When edit tracking is enabled, Roadie builds a preference profile from how the developer modifies generated content.

**Preference categories:**

| Category | Signal | Learned Preference |
|----------|--------|---------------------|
| Verbosity | Developer deletes verbose sections, keeps concise ones | Generate shorter sections |
| Code style | Developer rewrites arrow functions to named functions | Prefer named function style in generated code |
| Test depth | Developer adds edge case tests beyond what Roadie generates | Include more edge cases in test generation |
| Documentation depth | Developer removes JSDoc from internal functions | Only generate JSDoc for exported functions |
| Commit style | Developer rewrites commit messages to specific format | Match the format in future generation |

**Storage:** Preferences are stored as key-value pairs in the existing `developer_preferences` table:
```sql
INSERT INTO developer_preferences (key, value) VALUES
  ('pref.verbosity', '"concise"'),
  ('pref.functionStyle', '"named"'),
  ('pref.testDepth', '"comprehensive"'),
  ('pref.jsdocScope', '"exports_only"'),
  ('pref.commitFormat', '"conventional"');
```

**Integration:** The AdaptivePromptBuilder reads preferences and injects them as constraints:
```
Developer preferences (learned from edit history):
- Keep sections concise (developer removes verbose content 70% of the time)
- Use named functions, not arrow functions
- Generate JSDoc only for exported symbols
```

---

## What Phase 2.5 Does NOT Include

- No Copilot acceptance rate correlation (VS Code doesn't expose this to extensions)
- No cross-project learning (each project learns independently)
- No cloud-based model training (all learning is local, deterministic, rule-based)
- No automatic prompt rewriting (only recommendations - human approves template changes)
- No multi-user preference merging (solo developer only)

---

## Infrastructure Already Built (from Phase 1.5)

Phase 2.5 builds on data already being collected:

| Existing Component | What It Provides for Phase 2.5 |
|---|---|
| Edit Tracker (M21) | Before/after diffs of every developer modification to generated files |
| Learning Database (M23) | Snapshot storage, workflow history, section hashes |
| Section Manager hash tracking | Which sections were modified vs kept untouched |
| Codebase Dictionary (M24) | Entity metadata, modification history, dependency graph |
| `toContext()` with options | Token-budgeted context injection - already accepts `scope` parameter |
| Intent Classifier double-duty | LLM classification results already parsed and available for comparison |
| Workflow Engine outcome logging | Success/failure/duration/model tiers already recorded |

**What's genuinely new:** The analysis layer that reads this data and acts on it - quality scoring, adaptive prompts, weight tuning, step optimization, preference learning.

---

## Configuration

```json
{
  "roadie.adaptiveLearning": {
    "type": "boolean",
    "default": false,
    "description": "Enable adaptive learning. Requires editTracking and workflowHistory to also be true."
  },
  "roadie.adaptiveContextBudget": {
    "type": "number",
    "default": 500,
    "minimum": 100,
    "maximum": 2000,
    "description": "Maximum tokens for learned context injection into workflow prompts."
  },
  "roadie.classifierAutoTune": {
    "type": "boolean",
    "default": false,
    "description": "Allow the intent classifier to adjust pattern weights based on LLM correction history."
  }
}
```

**Dependency chain:** `adaptiveLearning` requires both `editTracking: true` and `workflowHistory: true`. If either dependency is false, adaptive learning silently degrades to static behavior.

---

## Milestones

### M29: Generation Quality Scoring Engine
**What:** Compute keep/edit/delete rates per section. Store in SQLite. Surface via `get_recommendations`.
**Depends on:** M21 (Edit Tracker), M23 (Learning Database)
**Estimated time:** 4-5 hours

### M30: Adaptive Prompt Builder
**What:** Extend PromptBuilder to inject learned context from workflow history, edit patterns, and codebase dictionary.
**Depends on:** M29, M24 (Codebase Dictionary)
**Estimated time:** 5-6 hours

### M31: Intent Classifier Auto-Tuning
**What:** Record classifier corrections. Compute weight adjustments. Store overrides in SQLite. Add reset command.
**Depends on:** M29, sufficient correction data (50+ workflow executions)
**Estimated time:** 4-5 hours

### M32: Workflow Step Optimizer
**What:** Track step performance. Auto-escalate chronically failing steps. Recommend prompt rewrites.
**Depends on:** M29
**Estimated time:** 3-4 hours

### M33: Developer Preference Learning
**What:** Extract preferences from edit patterns. Store as key-value pairs. Inject into adaptive prompts.
**Depends on:** M30 (Adaptive Prompt Builder)
**Estimated time:** 4-5 hours

---

## Build Order

```
M29 (Quality Scoring) <-- foundation, everything depends on this
    |
    +-- M30 (Adaptive Prompts) <-- uses quality scores + learned context
    |       |
    |       +-- M33 (Preference Learning) <-- feeds into adaptive prompts
    |
    +-- M31 (Classifier Tuning) <-- independent, needs correction data
    |
    +-- M32 (Step Optimizer) <-- independent, reads step performance data
```

---

## When to Build Phase 2.5

**NOT immediately after Phase 2.** Ship Phases 1-2 first, use Roadie on real projects for 2-4 weeks, then:

1. Check: does `editTracking` have enough data? (Need 50+ edit events across generated files)
2. Check: does `workflowHistory` have enough data? (Need 50+ workflow executions)
3. Check: are there classifier correction patterns? (Need 20+ LLM overrides of local classification)

If the data isn't there, Phase 2.5 has nothing to learn from. The learning algorithms are data-hungry - building them before data exists produces untested code.

**The critical insight:** Phase 2.5 is the only phase that CANNOT be spec'd fully before building. The quality thresholds (40% delete rate, 70% failure rate, etc.) are educated guesses. Real usage data will reveal the right thresholds. Expect to iterate.

---

## Key Technical Decisions

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| D12 | Learning approach | Rule-based, not ML | Deterministic, auditable, no training data requirements, works with small sample sizes |
| D13 | Weight tuning | Stored in SQLite, not code | Allows reset to factory defaults without code changes |
| D14 | Preference scope | Per-project, not global | Different projects have different conventions |
| D15 | Auto-optimization threshold | Conservative (70%+ failure before auto-escalate) | Avoid premature optimization that increases cost |
| D16 | Privacy | Only store prompt hashes, never prompts | Edit tracking stores diffs (which may contain code) - clearly documented in privacy model |
