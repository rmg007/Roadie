# 🔺 Model Selection Strategy

## Tier Hierarchy, Escalation, & Cost Awareness

Roadie uses three model tiers mapped to the VS Code Language Model API. Workflows always start at Tier 0 (free). Escalation only happens on failure.

---

## Model Tier Hierarchy

### **Tier 0: Free** (Cost: 0× premium requests)

**Available Models:**

- GPT-4.1 (or GPT-5 mini if available)
- Any free/included models from Copilot subscription

**Use Cases:**

- Initial attempt for every step
- Research, analysis, simple code generation
- Default for all workflow steps unless specified

**Characteristics:**

- Fast (typically <5s response time)
- Good enough for 80%+ of tasks
- No premium quota impact

**Default Assignment:**

- All bug-fix workflow steps except escalation
- Feature workflow steps 1, 4, 5, 7 (high-level planning/integration)
- Refactor steps except characterization test generation
- Review: Performance, Quality, Test Coverage, Standards (all Tier 0)
- Documentation, Dependency, Onboarding: All Tier 0

---

### **Tier 1: Standard** (Cost: 1× premium request per call)

**Available Models:**

- Claude Sonnet 4.6
- GPT-5.2
- Gemini 2.5 Pro

**Use Cases:**

- Escalation after Tier 0 fails
- More complex reasoning, multi-step logic
- Security-sensitive tasks (code review)

**Characteristics:**

- Slower than Tier 0 (typically 5-10s)
- Better reasoning, fewer errors
- Costs 1 premium request (Copilot Pro: 300/month)

**Default Assignment:**

- Bug-fix workflow: Step 2 (diagnose), escalation for step 3 (fix)
- Feature: Step 2 (plan approval), Step 3 parallel (layer planning), Step 6 (quality review)
- Refactor: Step 2 (characterization tests)
- Review: Security review (Tier 1 only; others Tier 0)
- Dependency: Step 3 (check breaking changes)

---

### **Tier 2: Premium** (Cost: 3× premium requests per call)

**Available Models:**

- Claude Opus 4.6

**Use Cases:**

- Last-resort escalation after Tier 1 fails
- Extremely difficult problems
- Rare in practice (<5% of workflow executions)

**Characteristics:**

- Slowest but most capable
- Deep reasoning, complex refactoring, architecture decisions
- Expensive (3 premium requests per call)

**Default Assignment:**

- Only on 3rd failure (after Tier 0 AND Tier 1 have failed)
- Not pre-assigned to any step; purely escalation

---

## Model Resolver Implementation

The `ModelResolver` maps tiers to actual models at runtime. **This is the canonical specification — copy verbatim.**

### Per-Tier Priority Table

Within a tier, models are tried in the order below. The first one available via `vscode.lm.selectChatModels()` is returned. `priority` is an exhaustive map (lower number = higher priority) across all tiers.

```tsx
// src/engine/model-priority.ts

/** Exhaustive priority map — every model named in any tier MUST appear here. */
export const MODEL_PRIORITY: Readonly<Record<string, number>> = {
  // Tier 2 (Premium) — tried first if tier === 'premium'
  'claude-opus-4.6':   0,
  // Tier 1 (Standard)
  'claude-sonnet-4.6': 10,
  'gpt-5.2':           11,
  'gemini-2.5-pro':    12,
  // Tier 0 (Free)
  'gpt-4.1':           20,
  'gpt-5-mini':        21,
} as const;

/** Tier → ordered preference list. Order in the array IS the fallback order within the tier. */
export const TIER_PREFERENCE: Readonly<Record<ModelTier, readonly string[]>> = {
  free:     ['gpt-4.1', 'gpt-5-mini'],
  standard: ['claude-sonnet-4.6', 'gpt-5.2', 'gemini-2.5-pro'],
  premium:  ['claude-opus-4.6'],
} as const;
```

### Error Taxonomy

```tsx
// src/engine/errors.ts

/** Thrown when no model is available for the requested tier AND fallback tiers have all been exhausted. */
export class ModelUnavailableError extends Error {
  readonly code = 'MODEL_UNAVAILABLE';
  readonly category = 'external' as const;
  readonly userFacing = true;

  constructor(
    public readonly requestedTier: ModelTier,
    public readonly triedModels: readonly string[],
  ) {
    super(
      `No language model available for tier '${requestedTier}'. ` +
      `Tried: ${triedModels.join(', ') || '(none)'}. ` +
      `Check your Copilot subscription or configure a direct API key.`,
    );
    this.name = 'ModelUnavailableError';
  }
}
```

Add `MODEL_UNAVAILABLE` to the corpus-wide error code list (see `08_Integration_and_Testing/Testing, Error Handling & Security.md`).

### Resolver Class

```tsx
// src/engine/model-resolver.ts

import * as vscode from 'vscode';
import { MODEL_PRIORITY, TIER_PREFERENCE } from './model-priority';
import { ModelUnavailableError } from './errors';
import type { ModelTier } from '../types';

export class ModelResolver {
  /**
   * Resolve a tier to a concrete `vscode.LanguageModelChat` instance.
   *
   * Algorithm:
   *   1. Enumerate available models via `vscode.lm.selectChatModels()`.
   *   2. For the requested tier, walk `TIER_PREFERENCE[tier]` in order.
   *   3. Return the first available model whose id contains the preference string.
   *   4. If none match, fall back to the next-lower tier (premium → standard → free).
   *   5. If even `free` has nothing, throw `ModelUnavailableError` with the complete list of models that were tried.
   *
   * Time budget: ≤ 50 ms (selectChatModels is cached by VS Code after first call).
   */
  async resolve(tier: ModelTier): Promise<vscode.LanguageModelChat> {
    const availableModels = await vscode.lm.selectChatModels();
    const tried: string[] = [];

    // 1. Try all preferences for the current tier in declared order.
    for (const preference of TIER_PREFERENCE[tier]) {
      tried.push(preference);
      const match = availableModels.find(m => m.id.includes(preference));
      if (match) return match;
    }

    // 2. Fall back to a lower tier if possible.
    if (tier === 'premium') return this.resolveWithCollectedAttempts('standard', tried);
    if (tier === 'standard') return this.resolveWithCollectedAttempts('free', tried);

    // 3. tier === 'free' and nothing matched: terminal failure.
    throw new ModelUnavailableError(tier, tried);
  }

  /** Internal helper — resolve and merge the `tried` list so the final error surfaces ALL attempted models. */
  private async resolveWithCollectedAttempts(
    tier: ModelTier,
    previouslyTried: readonly string[],
  ): Promise<vscode.LanguageModelChat> {
    try {
      return await this.resolve(tier);
    } catch (err) {
      if (err instanceof ModelUnavailableError) {
        // Rewrap with the full history
        throw new ModelUnavailableError(tier, [...previouslyTried, ...err.triedModels]);
      }
      throw err;
    }
  }
}
```

### Contract — What Callers Must Handle

1. `ModelUnavailableError` MUST be caught at the `WorkflowEngine.execute()` boundary and surfaced to the user via `vscode.window.showErrorMessage()` with "Check Subscription" and "Configure API Key" action buttons.
2. Callers MUST NOT retry the same tier on `ModelUnavailableError` — the resolver has already walked the full preference list and the fallback chain.
3. Callers MAY retry with `modelPreference: 'economy'` (forces Tier 0) if the user re-triggers the workflow after a premium failure.
4. `MODEL_PRIORITY` is exhaustive — every string in any `TIER_PREFERENCE[tier]` array MUST have a corresponding entry. Tests enforce this: see `model-priority.test.ts` below.

### Unit Tests (Mandatory)

```tsx
// test/engine/model-priority.test.ts
import { MODEL_PRIORITY, TIER_PREFERENCE } from '../../src/engine/model-priority';

describe('MODEL_PRIORITY exhaustiveness', () => {
  it('every model named in TIER_PREFERENCE has a priority entry', () => {
    for (const tier of Object.keys(TIER_PREFERENCE) as (keyof typeof TIER_PREFERENCE)[]) {
      for (const name of TIER_PREFERENCE[tier]) {
        expect(MODEL_PRIORITY[name]).toBeDefined();
      }
    }
  });

  it('priorities are unique', () => {
    const values = Object.values(MODEL_PRIORITY);
    expect(new Set(values).size).toBe(values.length);
  });
});

// test/engine/model-resolver.test.ts
describe('ModelResolver', () => {
  it('returns the first available preference within a tier', async () => {
    mockSelectChatModels.mockResolvedValue([
      { id: 'copilot-gpt-5.2' },
      { id: 'copilot-gemini-2.5-pro' },
    ]);
    const model = await resolver.resolve('standard');
    expect(model.id).toBe('copilot-gpt-5.2'); // gpt-5.2 appears earlier in TIER_PREFERENCE.standard
  });

  it('falls back from premium → standard → free', async () => {
    mockSelectChatModels.mockResolvedValue([{ id: 'copilot-gpt-4.1' }]);
    const model = await resolver.resolve('premium');
    expect(model.id).toBe('copilot-gpt-4.1');
  });

  it('throws ModelUnavailableError when nothing matches', async () => {
    mockSelectChatModels.mockResolvedValue([]);
    await expect(resolver.resolve('premium')).rejects.toBeInstanceOf(ModelUnavailableError);
  });

  it('error includes all tried models across tiers', async () => {
    mockSelectChatModels.mockResolvedValue([]);
    try {
      await resolver.resolve('premium');
      throw new Error('expected to throw');
    } catch (err) {
      expect(err).toBeInstanceOf(ModelUnavailableError);
      expect((err as ModelUnavailableError).triedModels).toEqual(expect.arrayContaining([
        'claude-opus-4.6',
        'claude-sonnet-4.6',
        'gpt-5.2',
        'gemini-2.5-pro',
        'gpt-4.1',
        'gpt-5-mini',
      ]));
    }
  });
});
```

---

## Escalation Logic

### When Escalation Triggers

**Escalation happens when a step produces output that fails validation:**

1. **Test Failure** (most common)
    - Step 4 in bug-fix: Tests don't pass
    - Step 4 in feature: Tests don't pass
    - Step 4 in refactor: Characterization tests fail
2. **Syntax Error**
    - Generated code doesn't compile/lint
    - Missing imports or undefined references
3. **Output Validation Failure**
    - Response doesn't match expected structure
    - LLM failed to follow instructions

### Escalation Sequence

```
Attempt 1 (Tier 0, Original Prompt):
  ↓ [success] → step complete, move to next step
  ↓ [failure] → attempt 2

Attempt 2 (Tier 0, Refined Prompt with Error Context):
  Include: "Your previous attempt failed: <error>. Try a different approach."
  ↓ [success] → step complete
  ↓ [failure] → attempt 3

Attempt 3 (Tier 1, Error + Diagnostic Logging):
  Higher tier, same prompt + request for debugging output
  ↓ [success] → step complete
  ↓ [failure] → attempt 4

Attempt 4 (Tier 1, Alternative Approach):
  Request completely different strategy
  ↓ [success] → step complete
  ↓ [failure] → attempt 5

Attempt 5 (Tier 2, Deep Analysis):
  Premium model, comprehensive analysis
  ↓ [success] → step complete
  ↓ [failure] → attempt 6

Attempt 6 (Developer Escalation):
  All attempts exhausted. Report to developer:
    - Problem diagnosed
    - All 5 attempts + their outputs
    - Recommendation for manual intervention
  Workflow state: PAUSED
  Developer decides next action
```

### Cost Example

```
Bug-fix workflow, step 3 (generate fix):

Attempt 1: Tier 0 (free) → fails
Attempt 2: Tier 0 (free) → fails
Attempt 3: Tier 1 (1 premium) → fails
Attempt 4: Tier 1 (1 premium) → fails
Attempt 5: Tier 2 (3 premium) → succeeds

Total cost: 5 premium requests (Tier 2: 3×, Tier 1: 2×1)

Budget impact: Copilot Pro (300 premium/month) = 60 complete workflows
```

---

## Per-Workflow Model Assignments

### Bug Fix Workflow

| Step | Name | Tier | Reason |
| --- | --- | --- | --- |
| 1 | Locate error source | Tier 0 | File search, simple grep |
| 2 | Diagnose root cause | **Tier 1** | More nuanced analysis |
| 3 | Generate and apply fix | Tier 0 → Tier 1/2 (escalation) | Attempt fix at Tier 0 first |
| 4 | Verify fix (run tests) | N/A | Shell command (no LLM) |
| 5 | Scan for sibling bugs | Tier 0 | Pattern search |
| 6 | Fix siblings (if found) | Tier 0 → Tier 1/2 (escalation) | Same as step 3 |
| 7 | Add regression guard | Tier 0 | Simple test generation |
| 8 | Generate summary | Tier 0 | Summarization |

---

### Feature Development Workflow

| Step | Name | Tier | Reason |
| --- | --- | --- | --- |
| 1 | Analyze requirements | Tier 0 | Parse user request |
| 2 | Present plan for approval | **Tier 0 → Tier 1** | Complex planning, needs quality |
| 3a | Database Agent | Tier 0 → Tier 1 | Schema changes are risky |
| 3b | Backend Agent | Tier 0 → Tier 1 | API design |
| 3c | Frontend Agent | Tier 0 | UI generation |
| 4 | Integrate layers | Tier 0 | Merging code |
| 5 | Run tests | N/A | Shell command |
| 6 | Quality review | **Tier 1** | Security + perf + quality |
| 7 | Generate commit messages | Tier 0 | Template-based |

---

### Code Review Workflow

| Pass | Name | Tier | Reason |
| --- | --- | --- | --- |
| 1 | Security Review | **Tier 1** | Security is critical, needs deeper analysis |
| 2 | Performance Review | Tier 0 | Pattern matching (N+1, complexity) |
| 3 | Code Quality Review | Tier 0 | Naming, duplication detection |
| 4 | Test Coverage Review | Tier 0 | Coverage gap analysis |
| 5 | Standards Review | Tier 0 | Project convention checking |

---

## Cost Budgeting

### Copilot Pro Budget (300 premium requests/month)

**Conservative estimate:**

- Bug-fix workflow: 5 premium avg (Tier 1 or escalation) × 60 workflows = 300 premium
- Feature workflow: 8 premium avg (multiple Tier 1 steps) × 30 workflows = 240 premium
- Code review: 1 premium (security pass only) × 100 reviews = 100 premium
- Other workflows (refactor, doc, etc.): 20 premium

**Total: ~660 premium requests/month**

**Mitigation:**

- Tier 0 by default everywhere possible
- Escalation only on failure
- Security review (only Tier 1 step) can be skipped with setting
- User can set `roadie.modelPreference` to `'economy'` (Tier 0 only, reduced quality)

---

## Configuration Options

### roadie.modelPreference

```json
{
  "roadie.modelPreference": "balanced" // default
}
```

**Values:**

- `"economy"` — Use Tier 0 only, no escalation (cheapest, lower quality)
- `"balanced"` — Tier 0 → escalate to Tier 1 on failure (default, good cost/quality)
- `"quality"` — Start at Tier 1 (expensive, highest quality)

**Effect:** Changes default model tier assignment, but escalation logic still applies.

---

## Fallback Behavior

Fallback is fully specified in the **Model Resolver Implementation** section above. The short version:

1. Walk `TIER_PREFERENCE[tier]` in declared order; return the first available match.
2. If no match, drop to the next-lower tier (premium → standard → free) and repeat.
3. If `free` also has no match, throw `ModelUnavailableError` with the full list of tried models.

### When `ModelUnavailableError` Is Thrown

The extension catches it at the `WorkflowEngine.execute()` boundary and presents:

```ts
const action = await vscode.window.showErrorMessage(
  err.message,
  'Check Copilot Subscription',
  'Configure API Key',
  'Dismiss',
);
```

- **"Check Copilot Subscription"** — opens `https://github.com/settings/copilot`.
- **"Configure API Key"** — opens the `roadie.apiKey` settings page (standalone mode).
- **"Dismiss"** — workflow transitions to `FAILED` state and pauses; developer can retry manually.

The workflow engine MUST NOT retry automatically on `ModelUnavailableError` — the resolver has already exhausted the preference list and the fallback chain.

---

## Testing Model Selection

```tsx
describe('ModelResolver', () => {
  it('should map free tier to GPT-4.1', async () => {
    const model = await resolver.resolve('free');
    expect(model.id).toMatch(/gpt-4\.1|gpt-5-mini/);
  });
  
  it('should fallback to free when standard unavailable', async () => {
    mockSelectChatModels.mockResolvedValue([{ id: 'gpt-4.1' }]);
    const model = await resolver.resolve('standard');
    expect(model.id).toBe('gpt-4.1');
  });
  
  it('should throw when no models available', async () => {
    mockSelectChatModels.mockResolvedValue([]);
    await expect(resolver.resolve('free')).rejects.toThrow();
  });
});
```

---

**Next:** Go to Workflow Definitions to see state machines & prompt templates.