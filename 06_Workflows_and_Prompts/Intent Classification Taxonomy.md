# 🔍 Intent Classification Taxonomy

## 8 Intent Types & Classification Strategy

The intent classifier recognizes 8 distinct intents. Each maps to a workflow or passthrough enrichment.

---

## Intent Catalog

### 1. **bug_fix**

**Purpose:** Fix an error or bug in the codebase  

**Workflow:** Bug Fix (8-step autonomous workflow)  

**Trigger Keywords:** "fix", "bug", "broken", "error", "not working", "crash", "failing", "500 error", "null pointer", "undefined is not"  

**Secondary Signals:** "login", "page", "endpoint", "stack trace", error messages  

**Confidence Threshold:** Primary signal + context → 0.5+; Multiple signals or stack trace → 0.8+  

**Example Prompts:**

- "Fix the login error on the settings page"
- "The API returns 500 after the deploy"
- "TypeError: Cannot read property 'id' of undefined"

**Escalation:** Step 4 (verify tests) failure → retry step 3 with test output + Tier 1

---

### 2. **feature**

**Purpose:** Add a new feature or capability  

**Workflow:** Feature Development (7 steps, human approval at step 2)  

**Trigger Keywords:** "add", "create", "build", "new feature", "implement", "make", "support", "enable"  

**Secondary Signals:** Specific feature names ("dark mode", "export", "search"), tech stack mentions  

**Confidence Threshold:** Primary + feature name → 0.8+; "add feature" alone → 0.7+  

**Example Prompts:**

- "Add a dark mode toggle to the settings page"
- "Build an export-to-PDF feature"
- "Implement user search with filters"

**Escalation:** If plan generation fails, remain at Tier 0 (plan is low-cost step)

---

### 3. **refactor**

**Purpose:** Restructure code without changing external behavior  

**Workflow:** Refactoring (5 steps with inner loop)  

**Trigger Keywords:** "refactor", "clean up", "restructure", "simplify", "extract", "reorganize", "optimize"  

**Secondary Signals:** Module/function names, mentions of "messy", "complex", "hard to test"  

**Confidence Threshold:** Primary → 0.8+; Module name + context → 0.7+  

**Example Prompts:**

- "Refactor the authentication module—it's gotten messy"
- "Extract the form validation logic into a separate module"
- "Simplify the data transformation pipeline"

**Escalation:** Step 2 (characterization tests) failure → Tier 1

---

### 4. **review**

**Purpose:** Review code changes for quality, security, performance  

**Workflow:** Code Review (5-pass parallel)  

**Trigger Keywords:** "review", "check", "audit", "analyze", "look at", "evaluate", "before I push", "any issues"  

**Secondary Signals:** "PR", "pull request", "commit", "changes", "my code"  

**Confidence Threshold:** Primary + context → 0.8+  

**Example Prompts:**

- "Review my changes before I push"
- "Check this PR for security issues"
- "Audit my code for performance problems"

**Escalation:** Security review uses Tier 1; others Tier 0

---

### 5. **document**

**Purpose:** Write or update documentation  

**Workflow:** Documentation (4 steps)  

**Trigger Keywords:** "document", "docs", "README", "API docs", "explain", "comment", "JSDoc", "write documentation"  

**Secondary Signals:** Module/function names, "how to", "what is"  

**Confidence Threshold:** Primary → 0.8+  

**Example Prompts:**

- "Document the API endpoints in this service"
- "Write a README for this module"
- "Add JSDoc comments to the auth module"

**Escalation:** None; documentation is low-cost

---

### 6. **dependency**

**Purpose:** Update, upgrade, or audit dependencies  

**Workflow:** Dependency Management (5 steps)  

**Trigger Keywords:** "update", "upgrade", "migrate", "dependency", "version", "package", "vulnerable", "CVE", "security audit"  

**Secondary Signals:** Specific package names ("React", "TypeScript"), "outdated", "breaking changes"  

**Confidence Threshold:** Primary + package name → 0.8+; Primary alone → 0.7+  

**Example Prompts:**

- "Update React to v19"
- "Run a security audit on our dependencies"
- "Upgrade TypeScript and fix any type errors"

**Escalation:** Breaking change detection → Tier 1

---

### 7. **onboard**

**Purpose:** Help someone understand the project  

**Workflow:** Onboarding (4 steps)  

**Trigger Keywords:** "onboard", "new to", "understand", "how does", "architecture", "where do I start", "walk me through", "explain the project", "starter task"  

**Secondary Signals:** "first task", "get up to speed", "what does this do"  

**Confidence Threshold:** Primary → 0.8+  

**Example Prompts:**

- "Help me understand the project architecture"
- "What does this codebase do?"
- "Walk me through the authentication flow"

**Escalation:** None; onboarding is low-cost

---

### 8. **general_chat**

**Purpose:** Fallback for non-workflow prompts  

**Workflow:** Passthrough (enriched with project context)  

**Trigger:** No primary signals match, OR multiple intents score similarly  

**Confidence Threshold:** Fallback (always 0.0)  

**Example Prompts:**

- "What's a good pattern for error handling?"
- "How do I write unit tests in Jest?"
- "Should we use React hooks or class components?"

**Behavior:** Prompt is enriched with project context (tech stack, patterns) and forwarded to LLM as single-turn request. Developer still benefits from Roadie selection even for casual questions.

---

## Classification Algorithm

### Phase 1: Local Classification

```
Algorithm: LocalClassifier.classify(prompt: string)

1. Normalize prompt: lowercase, trim whitespace, strip leading/trailing punctuation

2. For each intent in INTENT_PATTERNS (see matrix below):
   - For each pattern entry:
     - Test entry.regex against the normalized prompt
     - If matches: add entry.weight to the intent's running score
     - Add entry.label to matched signals list
   - Cap intent score at 1.0

3. Apply NEGATIVE_SIGNALS (see matrix below):
   - If a negative signal matches, apply its negative weight to the top-scoring intent

4. Find intent with highest score

5. Ambiguity check:
   - Sort scores descending
   - If scores[0] - scores[1] < 0.3: cap confidence at 0.6, set requiresLLM=true
   - If scores[0] >= 0.7: return with confidence=scores[0], requiresLLM=false
   - If scores[0] < 0.3: return { intent: 'general_chat', confidence: 0.1, requiresLLM: true }

6. Return ClassificationResult
```

### Exact Pattern Weight Matrix

**This is the canonical definition for `src/classifier/intent-patterns.ts`.** Copy verbatim — do not infer weights.

```ts
// src/classifier/intent-patterns.ts

export interface IntentPattern {
  regex: RegExp;
  weight: number;   // additive score contribution per match (negative = reduces confidence)
  label: string;    // for debugging / matched signals output
}

export const INTENT_PATTERNS: Record<string, IntentPattern[]> = {
  bug_fix: [
    { regex: /\bfix\b/i,                                    weight: 0.35, label: 'keyword:fix' },
    { regex: /\bbug\b/i,                                    weight: 0.35, label: 'keyword:bug' },
    { regex: /\bbroken\b/i,                                 weight: 0.30, label: 'keyword:broken' },
    { regex: /\berror\b/i,                                  weight: 0.25, label: 'keyword:error' },
    { regex: /\bnot working\b/i,                            weight: 0.30, label: 'keyword:not-working' },
    { regex: /\bcrash(ing|es)?\b/i,                         weight: 0.35, label: 'keyword:crash' },
    { regex: /\bfailing\b/i,                                weight: 0.30, label: 'keyword:failing' },
    { regex: /\b500\b/,                                     weight: 0.20, label: 'signal:500-error' },
    { regex: /null pointer|undefined is not|cannot read/i,  weight: 0.40, label: 'signal:error-message' },
    { regex: /TypeError:|ReferenceError:|SyntaxError:/,     weight: 0.45, label: 'signal:runtime-error-type' },
    { regex: /at \/[\w/.-]+:\d+/,                          weight: 0.40, label: 'signal:stack-trace' },
    { regex: /\bstack trace\b/i,                            weight: 0.35, label: 'signal:stack-trace-mention' },
  ],

  feature: [
    { regex: /\badd\b/i,                                    weight: 0.30, label: 'keyword:add' },
    { regex: /\bcreate\b/i,                                 weight: 0.25, label: 'keyword:create' },
    { regex: /\bbuild\b/i,                                  weight: 0.20, label: 'keyword:build' },
    { regex: /\bnew feature\b/i,                            weight: 0.40, label: 'keyword:new-feature' },
    { regex: /\bimplement\b/i,                              weight: 0.35, label: 'keyword:implement' },
    { regex: /\bmake\b.*\bwork\b/i,                         weight: 0.20, label: 'keyword:make-work' },
    { regex: /\bsupport\b/i,                                weight: 0.20, label: 'keyword:support' },
    { regex: /\benable\b/i,                                 weight: 0.20, label: 'keyword:enable' },
    { regex: /dark mode|export|search|filter|pagination/i,  weight: 0.25, label: 'signal:feature-name' },
  ],

  refactor: [
    { regex: /\brefactor\b/i,                               weight: 0.45, label: 'keyword:refactor' },
    { regex: /\bclean\s*up\b/i,                             weight: 0.35, label: 'keyword:clean-up' },
    { regex: /\brestructure\b/i,                            weight: 0.40, label: 'keyword:restructure' },
    { regex: /\bsimplify\b/i,                               weight: 0.35, label: 'keyword:simplify' },
    { regex: /\bextract\b/i,                                weight: 0.30, label: 'keyword:extract' },
    { regex: /\breorganize\b/i,                             weight: 0.35, label: 'keyword:reorganize' },
    { regex: /\boptimize\b/i,                               weight: 0.25, label: 'keyword:optimize' },
    { regex: /\bmessy\b|\bcomplex\b|\bhard to test\b/i,    weight: 0.20, label: 'signal:quality-complaint' },
  ],

  review: [
    { regex: /\breview\b/i,                                 weight: 0.45, label: 'keyword:review' },
    { regex: /\baudit\b/i,                                  weight: 0.40, label: 'keyword:audit' },
    { regex: /\banalyze\b/i,                                weight: 0.30, label: 'keyword:analyze' },
    { regex: /\bevaluate\b/i,                               weight: 0.25, label: 'keyword:evaluate' },
    { regex: /\blook at\b/i,                                weight: 0.25, label: 'keyword:look-at' },
    { regex: /\bbefore I push\b/i,                          weight: 0.40, label: 'signal:before-push' },
    { regex: /\bany issues\b/i,                             weight: 0.30, label: 'signal:any-issues' },
    { regex: /\bPR\b|\bpull request\b/i,                    weight: 0.25, label: 'signal:PR' },
    { regex: /\bmy (code|changes)\b/i,                      weight: 0.20, label: 'signal:my-code' },
  ],

  document: [
    { regex: /\bdocument\b/i,                               weight: 0.45, label: 'keyword:document' },
    { regex: /\bdocs?\b/i,                                  weight: 0.35, label: 'keyword:docs' },
    { regex: /\bREADME\b/,                                  weight: 0.40, label: 'signal:README' },
    { regex: /\bAPI docs?\b/i,                              weight: 0.40, label: 'signal:API-docs' },
    { regex: /\bJSDoc\b/i,                                  weight: 0.40, label: 'signal:JSDoc' },
    { regex: /\bwrite documentation\b/i,                    weight: 0.45, label: 'keyword:write-documentation' },
    { regex: /\bcomments?\b/i,                              weight: 0.20, label: 'keyword:comments' },
    { regex: /\bexplain\b/i,                                weight: 0.15, label: 'keyword:explain' },
  ],

  dependency: [
    { regex: /\bupgrade\b/i,                                weight: 0.35, label: 'keyword:upgrade' },
    { regex: /\bmigrate\b/i,                                weight: 0.30, label: 'keyword:migrate' },
    { regex: /\bdependenc(y|ies)\b/i,                       weight: 0.40, label: 'keyword:dependency' },
    { regex: /\bpackage\b/i,                                weight: 0.20, label: 'keyword:package' },
    { regex: /\bvulnerab|CVE\b/i,                           weight: 0.40, label: 'signal:vulnerability' },
    { regex: /\boutdated\b/i,                               weight: 0.35, label: 'signal:outdated' },
    { regex: /\bbreaking changes?\b/i,                      weight: 0.30, label: 'signal:breaking-change' },
    { regex: /\bsecurity audit\b/i,                         weight: 0.40, label: 'signal:security-audit' },
    { regex: /\bReact\b|\bTypeScript\b|\bPrisma\b|\bNext\.js\b|\bExpress\b/, weight: 0.20, label: 'signal:package-name' },
  ],

  onboard: [
    { regex: /\bonboard\b/i,                                weight: 0.45, label: 'keyword:onboard' },
    { regex: /\bnew to\b/i,                                 weight: 0.35, label: 'keyword:new-to' },
    { regex: /\bunderstand\b/i,                             weight: 0.20, label: 'keyword:understand' },
    { regex: /\bhow does\b/i,                               weight: 0.20, label: 'keyword:how-does' },
    { regex: /\barchitecture\b/i,                           weight: 0.25, label: 'keyword:architecture' },
    { regex: /\bwhere do I start\b/i,                       weight: 0.40, label: 'keyword:where-start' },
    { regex: /\bwalk me through\b/i,                        weight: 0.40, label: 'signal:walk-through' },
    { regex: /\bexplain the project\b/i,                    weight: 0.45, label: 'signal:explain-project' },
    { regex: /\bstarter task\b|\bfirst task\b/i,            weight: 0.35, label: 'signal:starter-task' },
    { regex: /\bget up to speed\b/i,                        weight: 0.40, label: 'signal:up-to-speed' },
  ],

  // general_chat is the fallback — no patterns. It is returned when no other intent scores >= 0.3.
};

// Negative signals: if these match, subtract weight from the top-scoring intent
export const NEGATIVE_SIGNALS: IntentPattern[] = [
  { regex: /\bdon'?t fix\b|\bdo not fix\b/i,        weight: -0.40, label: 'negative:dont-fix' },
  { regex: /\bdon'?t document\b/i,                   weight: -0.40, label: 'negative:dont-document' },
  { regex: /\bnot a bug\b|\bexpected behavior\b/i,   weight: -0.30, label: 'negative:not-a-bug' },
];

// Canonical confidence thresholds — every workflow and every test MUST import this
// constant rather than hardcoding floats. The values are frozen via `as const` so
// downstream code receives literal types (e.g. `0.80`, not `number`).
export const CONFIDENCE_THRESHOLDS = {
  /** Single primary signal matched → confidence returned by classifier. */
  primaryOnly: 0.80,
  /** Primary + at least one secondary signal matched → highest local confidence. */
  primaryPlusSecondary: 0.90,
  /** Two or more intents scored within this delta of each other → confidence is capped. */
  ambiguousDelta: 0.30,
  /** Cap applied when multiple intents are within `ambiguousDelta` of each other. */
  ambiguousCap: 0.60,
  /** Below this score, no intent is recognized → classifier returns general_chat with this confidence. */
  unknown: 0.10,
  /** A negative signal matches the top intent → confidence collapses to this floor. */
  negativeOverride: 0.10,
  /** Minimum confidence at which the classifier is allowed to skip LLM fallback. */
  requiresLLMBelow: 0.70,
  /** LLM fallback always returns this confidence when it produces a classification. */
  llmClassification: 0.85,
} as const;
```

**Latency Requirement:** <10ms (no I/O — all patterns evaluated synchronously in a single `for` loop)



### Phase 2: LLM Classification (if requiresLLM=true)

Append to system prompt:

```
Before responding, classify this request as one of:
  - bug_fix: Fix an error or bug
  - feature: Add a new feature
  - refactor: Restructure code
  - review: Review code changes
  - document: Write/update documentation
  - dependency: Update dependencies
  - onboard: Understand the project
  - general_chat: General question not matching above

Respond with a JSON block FIRST, before your main response:
{"intent": "<intent>", "reasoning": "<brief explanation>"}

Then provide your main response.
```

**Parsing:**

1. Extract first JSON block from response: `{...}` 
2. Validate against schema: intent must be one of 8 types
3. Return ClassificationResult with intent + high confidence (0.85+)
4. Continue with main response (don't discard it)

**Cost:** 1 LLM call, counted against budget, piggybacked on first workflow step

---

## Confidence Scoring Rules

| Scenario | Confidence | RequiresLLM |
| --- | --- | --- |
| Single primary signal only | 0.80 | false |
| Primary + secondary signals combined | 0.90 | false |
| Multiple intents within 0.3 of each other | 0.60 (capped) | true |
| Highest score < 0.3 (no recognizable signal) | 0.10 | true (→ general_chat) |
| Negative signals override top intent | 0.10 | false (→ general_chat) |

> **Canonical source for threshold values:** `CONFIDENCE_THRESHOLDS` in `06_Workflows_and_Prompts/Intent Classification Taxonomy.md`.
> When asserting in tests, use the values from that constant — do not hardcode floats from this table.

---

## Edge Cases

### Multiple Intent Signals

```
Prompt: "Refactor the authentication module, then write docs for it"

Scores: refactor (0.75), document (0.65)
Difference: 0.1 < 0.3 threshold
Result: Confidence capped at 0.6, requiresLLM=true

LLM resolves: likely "refactor" (primary action), suggests documentation as follow-up
```

### Ambiguous Project Context

```
Prompt: "Update the login page"

Could be: feature (add login feature) OR bug_fix (fix broken login)
Scores: feature (0.55), bug_fix (0.5)
Result: requiresLLM=true, LLM asks for clarification or picks most likely based on context
```

### Stack Traces & Error Messages

```
Prompt contains: "TypeError: Cannot read property 'user' of undefined at /src/auth.ts:42"

Scores: bug_fix (0.95) [strong signal: error message + stack trace]
Result: intent=bug_fix, confidence=0.95, requiresLLM=false
```

---

## Testing Intent Classification

### Test Suite Structure

```tsx
describe('IntentClassifier', () => {
  describe('bug_fix intent', () => {
    it('should detect "fix the login error"', () => {...})
    it('should detect "500 error on profile page"', () => {...})
    it('should detect stack trace', () => {...})
    // 10+ cases
  })
  
  describe('feature intent', () => {
    // 10+ cases
  })
  
  // ... per intent
  
  describe('edge cases', () => {
    it('should set requiresLLM=true for ambiguous prompts', () => {...})
    it('should handle negative signals', () => {...})
    it('should achieve <10ms latency', () => {...})
  })
})
```

### Success Criteria

- ≥90% accuracy on the reference dataset below (100 prompts)
- All 8 intents correctly identified
- Ambiguous cases (within `CONFIDENCE_THRESHOLDS.ambiguousDelta`) produce `requiresLLM=true`
- <10ms latency for local classifier
- Fallback to `general_chat` for unknown prompts

---

## Canonical Reference Test Dataset (100 prompts)

**Purpose:** This is the frozen dataset used to measure classifier accuracy. The Definition of Done for Step 5 of `Module Build Order & Verification.md` requires ≥ 90 correct classifications out of 100 using `INTENT_PATTERNS`, `NEGATIVE_SIGNALS`, and `CONFIDENCE_THRESHOLDS` verbatim.

**File to create:** `test/fixtures/intent-classification/dataset.json`

**Usage:**

```tsx
import dataset from './fixtures/intent-classification/dataset.json';
import type { IntentType } from '../src/types';
import { IntentClassifier } from '../src/classifier/intent-classifier';

describe('IntentClassifier accuracy', () => {
  const classifier = new IntentClassifier();

  it('achieves ≥90% accuracy on the reference dataset', () => {
    let correct = 0;
    for (const row of dataset) {
      const result = classifier.classify(row.prompt);
      if (result.intent === row.expectedIntent) correct++;
    }
    expect(correct / dataset.length).toBeGreaterThanOrEqual(0.90);
  });

  it('every row has the expected confidence within ±0.10', () => {
    for (const row of dataset) {
      const result = classifier.classify(row.prompt);
      if (result.intent === row.expectedIntent) {
        expect(Math.abs(result.confidence - row.expectedConfidence)).toBeLessThanOrEqual(0.10);
      }
    }
  });
});
```

**Dataset contents** — copy verbatim into `test/fixtures/intent-classification/dataset.json`:

```json
[
  { "prompt": "Fix the login bug",                                                            "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "The checkout page is broken when the cart is empty",                           "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "500 error on profile endpoint",                                                "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "TypeError: Cannot read property 'user' of undefined at /src/auth.ts:42",       "expectedIntent": "bug_fix",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "App crashing on startup since upgrading Node",                                 "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "ReferenceError: process is not defined in the browser",                        "expectedIntent": "bug_fix",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Getting a null pointer exception in the user serializer",                      "expectedIntent": "bug_fix",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Login stopped working after the last deploy",                                  "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Tests are failing after I merged main",                                        "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "There's a stack trace in the logs — can you look?",                            "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "fix the race condition in the queue worker",                                   "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "The avatar upload returns 413 Payload Too Large",                              "expectedIntent": "bug_fix",    "expectedConfidence": 0.80, "requiresLLM": false },

  { "prompt": "Add dark mode to the settings page",                                           "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Implement a password reset flow",                                              "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Create an export to CSV button on the reports page",                           "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Build a new admin dashboard with user management",                             "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "I want to add pagination to the search results",                               "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Support Google Sign-In as an alternative to email/password",                   "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Add a filter dropdown for the product list",                                   "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "New feature: allow users to set a display name",                               "expectedIntent": "feature",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Enable webhooks for the billing module",                                       "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Make the sidebar collapsible",                                                 "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Add a two-factor auth prompt on sensitive actions",                            "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Create a command palette shortcut for \"duplicate file\"",                     "expectedIntent": "feature",    "expectedConfidence": 0.80, "requiresLLM": false },

  { "prompt": "Refactor the authentication module",                                           "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Clean up the controller layer — it has grown too messy",                       "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Restructure the routes file, it's getting hard to test",                       "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Simplify the order total calculation",                                         "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Extract the retry logic into a reusable helper",                               "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Reorganize the utils directory by domain",                                     "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Optimize the N+1 queries in OrderService",                                     "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "This function is too complex, please simplify without changing the API",       "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Split the monolithic user service into smaller modules",                       "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Refactor to remove the circular dependency between auth and billing",          "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Clean up the duplicated validation logic across controllers",                  "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Restructure the reducers using slices",                                        "expectedIntent": "refactor",   "expectedConfidence": 0.80, "requiresLLM": false },

  { "prompt": "Review my changes before I push",                                              "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Audit the OrderController for security issues",                                "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Evaluate the new caching layer for correctness",                               "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Look at my PR #247 and tell me what's wrong",                                  "expectedIntent": "review",     "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Before I push, any issues you can spot?",                                      "expectedIntent": "review",     "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Can you review the diff on the billing rewrite?",                              "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Analyze the changes in the last commit for regressions",                       "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Do a quick review of my code before I open the PR",                            "expectedIntent": "review",     "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Security review of the new auth middleware",                                   "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "I want a review of my pull request",                                           "expectedIntent": "review",     "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Can you audit the Stripe webhook handler for race conditions?",                "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Review my changes to the GraphQL resolvers",                                   "expectedIntent": "review",     "expectedConfidence": 0.80, "requiresLLM": false },

  { "prompt": "Document the public API of UserService",                                       "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Write a README for the payments package",                                      "expectedIntent": "document",   "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Add JSDoc comments to the math helpers",                                       "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Write documentation for the new onboarding flow",                              "expectedIntent": "document",   "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Generate API docs from the OpenAPI spec",                                      "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Document how the retry queue works",                                           "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Add inline comments explaining the permission algorithm",                      "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Produce a user-facing docs page for feature flags",                            "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Write a README.md at the repo root",                                           "expectedIntent": "document",   "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Document the environment variables we expect in production",                   "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Create API docs for the /users endpoints",                                     "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Docs for the new SDK please",                                                  "expectedIntent": "document",   "expectedConfidence": 0.80, "requiresLLM": false },

  { "prompt": "Upgrade React to version 19",                                                  "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Migrate from Prisma 4 to Prisma 5",                                            "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Check our dependencies for outdated packages",                                 "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Do a security audit of our npm dependencies",                                  "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Which of our packages have CVEs?",                                             "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Bump TypeScript to the latest stable",                                         "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Upgrade Next.js and fix any breaking changes",                                 "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Update our dependencies with patch-level fixes",                               "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Migrate Express to Fastify",                                                   "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Run a vulnerability scan against package-lock.json",                           "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Our package-lock is outdated, refresh it",                                     "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Move from Webpack 4 to Vite",                                                  "expectedIntent": "dependency", "expectedConfidence": 0.80, "requiresLLM": false },

  { "prompt": "I'm new to this codebase, where do I start?",                                  "expectedIntent": "onboard",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Walk me through the architecture of this project",                             "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Give me a starter task to get up to speed",                                    "expectedIntent": "onboard",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Explain the project structure to a new engineer",                              "expectedIntent": "onboard",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Can you onboard me to this repository?",                                       "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "How does the auth layer work?",                                                "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "I'm new to this stack — how does the data flow work?",                         "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "First task for an intern would be great",                                      "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Help me understand how billing talks to Stripe",                               "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "Explain the project so I can get started",                                     "expectedIntent": "onboard",    "expectedConfidence": 0.90, "requiresLLM": false },
  { "prompt": "Walk me through the payment workflow end to end",                              "expectedIntent": "onboard",    "expectedConfidence": 0.80, "requiresLLM": false },
  { "prompt": "I just joined — where do I start?",                                             "expectedIntent": "onboard",    "expectedConfidence": 0.90, "requiresLLM": false },

  { "prompt": "Don't fix it, just tell me what's wrong",                                      "expectedIntent": "general_chat","expectedConfidence": 0.10, "requiresLLM": false },
  { "prompt": "That's not a bug, it's expected behavior — can you explain it?",               "expectedIntent": "general_chat","expectedConfidence": 0.10, "requiresLLM": false },
  { "prompt": "What is a reverse proxy?",                                                     "expectedIntent": "general_chat","expectedConfidence": 0.10, "requiresLLM": true  },
  { "prompt": "Tell me a joke",                                                               "expectedIntent": "general_chat","expectedConfidence": 0.10, "requiresLLM": true  },

  { "prompt": "Refactor the authentication module and write docs for it",                     "expectedIntent": "refactor",   "expectedConfidence": 0.60, "requiresLLM": true  },
  { "prompt": "Update the login page",                                                        "expectedIntent": "feature",    "expectedConfidence": 0.60, "requiresLLM": true  },
  { "prompt": "Clean up dependencies and fix the failing tests",                              "expectedIntent": "dependency", "expectedConfidence": 0.60, "requiresLLM": true  },
  { "prompt": "Review and refactor the billing module",                                       "expectedIntent": "review",     "expectedConfidence": 0.60, "requiresLLM": true  }
]
```

**Dataset accounting (must match counts in the success criteria):**

| Intent | Count |
| --- | --- |
| bug_fix | 12 |
| feature | 12 |
| refactor | 12 |
| review | 12 |
| document | 12 |
| dependency | 12 |
| onboard | 12 |
| general_chat | 4 (2 clean + 2 negative-override) |
| ambiguous (requiresLLM=true) | 4 |
| **Total** | **100** (72 single-intent + 4 general_chat + 4 ambiguous + 20 reserved for Phase 1.5 growth — Phase 1 test uses the 92 primary rows and MUST achieve ≥ 83/92 correct) |

> **Note:** The 100-row file above has 92 deterministic rows. The remaining 8 slots are reserved — Phase 1.5 will expand the dataset by 8 rows covering the new Phase 1.5 workflows. Phase 1 tests MUST run against the 92 rows currently present.

---

**Next:** Go to Model Selection Strategy for tier hierarchy and escalation logic.