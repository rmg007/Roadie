# 💼 Module-Specific Implementation Guides

## Deep Dive Specs for Each of the 14 Phase 1 Modules

**Purpose:** Detailed specs that Claude Code can reference while implementing each module.  

**Organization:** One section per module, in build order (M0-M13).  

**Format:** Interface, pseudocode algorithms, edge cases, testing strategy.

---

## M0: types.ts

### Public Interface

```tsx
// Export all 50+ types
// See "Shared TypeScript Interfaces" page for complete listing
```

### Special Considerations

- **No Logic:** This file contains types ONLY. Zero implementation.
- **Documentation:** Every type must have JSDoc explaining its purpose and usage.
- **Imports:** Should import nothing (except TypeScript standard library if needed).
- **Circular Imports:** Impossible by design (lowest-level file).

### Test Strategy

No tests for types.ts (it's just types). However, verify:

- Builds cleanly: `npm run build`
- No TypeScript errors: `npx tsc --noEmit`
- All exports available: Can import all types in a test file

---

## M0: extension.ts

### Core Logic Pseudocode

```
function activate(context: vscode.ExtensionContext):
  logger.info("Roadie activating...")
  
  // Initialize dependency injection container
  container = new Container()
  container.registerSingleton('projectModel', ProjectModel)
  container.registerSingleton('workflowEngine', WorkflowEngine)
  container.registerSingleton('classifier', IntentClassifier)
  // ... register all modules
  
  // Register Chat Participant
  const handler = new ChatParticipantHandler(container)
  vscode.chat.createChatParticipant('roadie', handler)
  
  // Register status bar
  statusBar = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right)
  statusBar.text = "$(sync~spin) Roadie"
  statusBar.show()
  
  // Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand('roadie.init', () => initialize())
  )
  
  logger.info("Roadie activated")
end

function deactivate():
  logger.info("Roadie deactivating...")
  // Cleanup: dispose all subscriptions, close connections, etc.
end
```

### Error Handling

```
if activation throws:
  log error with context
  show vscode.window.showErrorMessage("Roadie failed to activate: ...")
  FAIL GRACEFULLY (don't crash VS Code)
end
```

### Files

- `src/extension.ts` (~150 lines)
- `src/extension.test.ts` (~100 lines integration tests)

---

## M0: container.ts

### Dependency Injection Pattern

```tsx
interface IContainer {
  registerSingleton<T>(key: string, factory: () => T): void;
  resolve<T>(key: string): T;
}

class Container implements IContainer {
  private instances = new Map<string, any>();
  
  registerSingleton<T>(key: string, factory: () => T): void {
    this.instances.set(key, null); // Lazy init marker
  }
  
  resolve<T>(key: string): T {
    if (!this.instances.has(key)) {
      throw new Error(`Unknown dependency: ${key}`);
    }
    if (this.instances.get(key) === null) {
      // Create instance on first access
      this.instances.set(key, factoryMap[key]());
    }
    return this.instances.get(key) as T;
  }
}
```

### Registration in extension.ts

```tsx
const container = new Container();
container.registerSingleton('projectModel', () => new ProjectModel());
container.registerSingleton('workflowEngine', () => 
  new WorkflowEngine(container.resolve('projectModel'))
);
// ... etc for all 10+ modules
```

---

## M1: model-resolver.ts

### Core Algorithm

```
async function resolveModel(tier: ModelTier): Promise<LanguageModelChat> {
  // Get available models from VS Code LM API
  const available = await vscode.lm.selectChatModels();
  
  // Map tier to model names
  const tierMap = {
    'free': ['gpt-5-mini', 'gpt-4.1'],
    'standard': ['claude-sonnet-4.6', 'gpt-5.2'],
    'premium': ['claude-opus-4.6']
  };
  
  // Find first matching model
  for (const modelName of tierMap[tier]) {
    const match = available.find(m => m.id.includes(modelName));
    if (match) return match;
  }
  
  // Tier not available, try lower tier
  if (tier === 'standard') return resolveModel('free');
  if (tier === 'premium') return resolveModel('standard');
  
  // No models available at all
  throw new Error('No language models available');
end
```

### Edge Cases

1. **Model string matching:** Use `.includes()` not exact match (IDs vary)
2. **Unavailable tier:** Recursively fall back to lower tier
3. **No models:** Throw with clear error message
4. **Caching:** Do not cache results (models availability can change)

### Files

- `src/engine/model-resolver.ts` (~120 lines)
- `src/engine/model-resolver.test.ts` (~150 lines, 8+ tests)

---

## M2: intent-classifier.ts

### Two-Tier Classification Flow

```jsx
function classify(prompt: string): ClassificationResult {
  // Tier 1: Local (instant, zero cost)
  const local = classifyLocal(prompt);
  
  if (local.confidence >= 0.7) {
    return local; // High confidence, don't escalate
  }
  
  // Low confidence—will need LLM
  return {
    intent: local.intent,
    confidence: local.confidence,
    signals: local.signals,
    requiresLLM: true // Flag for ChatParticipantHandler
  };
end

// Called by ChatParticipantHandler to get the prefix to prepend to the system prompt
// when LLM classification is needed (requiresLLM === true)
function getClassificationPromptPrefix(): string {
  return `
    Before responding, classify this request as one of 8 intent types.
    Respond with JSON FIRST on its own line:
    {"intent": "bug_fix|feature|refactor|review|document|dependency|onboard|general_chat", "reasoning": "..."}
    Then proceed with your response.
  `;
end

// Called by ChatParticipantHandler to parse classification from the LLM response
// This is a DOUBLE-DUTY call — one LLM invocation serves both classification AND response
function parseClassification(responseText: string): ClassificationResult | null {
  // Extract JSON from response
  const jsonMatch = responseText.match(/\{[\s\S]*?\}/);
  if (!jsonMatch) return null;
  
  try {
    const parsed = JSON.parse(jsonMatch[0]);
    if (!isValidIntentType(parsed.intent)) return null;
    return {
      intent: parsed.intent,
      confidence: 0.85, // LLM classified—high confidence
      signals: [],
      requiresLLM: false
    };
  } catch {
    return null; // Parse failed, fallback to general_chat
  }
end
```

### Local Classifier Pseudocode

```
function classifyLocal(prompt: string): ClassificationResult {
  const scores = {};
  
  for (const intent in intentPatterns) {
    scores[intent] = 0;
    
    for (const pattern in intentPatterns[intent]) {
      const weight = pattern.weight; // 0.2-0.4
      const regex = pattern.regex;
      
      if (regex.test(prompt)) {
        scores[intent] += weight;
      }
    }
  }
  
  // Find top intent
  const sorted = Object.entries(scores).sort((a, b) => b[1] - a[1]);
  const topIntent = sorted[0][0];
  const topScore = sorted[0][1];
  
  // Check for ambiguity
  if (sorted.length > 1 && (sorted[1][1] > topScore - 0.3)) {
    // Multiple intents scoring close—ambiguous
    return {intent: topIntent, confidence: 0.6, requiresLLM: true, ...};
  }
  
  if (topScore < 0.3) {
    // No strong match—fallback
    return {intent: 'general_chat', confidence: 0.1, ...};
  }
  
  return {intent: topIntent, confidence: topScore, requiresLLM: false, ...};
end
```

### Test Cases

- 100+ prompt examples covering all 8 intents
- Test accuracy: >= 90%
- Test latency: < 10ms
- Test ambiguity handling
- Test negative signals
- Test LLM fallback

---

## M3: workflow-engine.ts

### State Machine Pseudocode

```
class WorkflowEngine {
  async execute(definition: WorkflowDefinition, context: WorkflowContext): Promise<WorkflowResult> {
    const execution = new WorkflowExecution(definition.id, context);
    execution.state = 'RUNNING';
    
    const results = [];
    let currentStepIndex = 0;
    
    while (currentStepIndex < definition.steps.length) {
      const step = definition.steps[currentStepIndex];
      
      if (context.cancellationToken.isCancellationRequested) {
        execution.state = 'CANCELLED';
        break;
      }
      
      try {
        // Execute step (sequential or parallel)
        let stepResult;
        if (step.type === 'sequential' || step.type === 'conditional') {
          stepResult = await this.stepExecutor.execute(step, context);
        } else if (step.type === 'parallel') {
          stepResult = await this.executeParallel(step.branches, context);
        }
        
        results.push(stepResult);
        
        // Advance to next step
        if (step.type === 'conditional') {
          const nextStepId = step.condition(stepResult);
          currentStepIndex = definition.steps.findIndex(s => s.id === nextStepId);
        } else {
          currentStepIndex++;
        }
      } catch (error) {
        // Step failed—let StepExecutor handle retry/escalation
        // StepExecutor throws only after max retries
        execution.state = 'FAILED';
        return {state: 'FAILED', stepResults: results, error: error.message};
      }
    }
    
    execution.state = 'COMPLETED';
    return {state: 'COMPLETED', stepResults: results, ...};
  }
  
  async executeParallel(branches: WorkflowStep[], context: WorkflowContext): Promise<StepResult[]> {
    const promises = branches.map(branch => this.stepExecutor.execute(branch, context));
    const results = await Promise.allSettled(promises);
    
    return results.map((r, i) => 
      r.status === 'fulfilled' ? r.value : {stepId: branches[i].id, status: 'failed', ...}
    );
  }
}
```

### Stream Progress Updates

```
while executing steps:
  after each step completes:
    context.chatResponseStream.markdown(`✓ ${step.name}`);
  after step fails or pauses:
    context.chatResponseStream.markdown(`❌ ${step.name}: ${error}`);
end
```

---

## M4-M5: Agent Spawner, Project Model

[See detailed specs in dedicated pages for Agent Spawner and Project Model]

---

## M6: bug-fix.ts (Workflow Definition)

### Declarative Definition

```tsx
const bugFixWorkflow: WorkflowDefinition = {
  id: 'bug_fix',
  name: 'Bug Fix Workflow',
  steps: [
    {
      id: 'step-1',
      name: 'Locate Error Source',
      type: 'sequential',
      agentRole: 'diagnostician',
      modelTier: 'free',
      toolScope: 'research',
      promptTemplate: templates.bugFixStep1,
      timeoutMs: 30000,
      maxRetries: 3
    },
    // ... 7 more steps
  ]
};
```

### No Implementation Logic

Workflow definitions are DECLARATIVE ONLY. No logic—just data.

The WorkflowEngine interprets and executes them.

---

## Remaining Modules

M7-M13: (File Generator, Refactor Workflow, Review Workflow, Feature Workflow, Documentation, Dependency, Onboarding, Configuration, Polish)

Detailed specs will be provided on request. Follow the patterns established in M0-M6:

- Clear pseudocode algorithms
- Edge case handling
- Error handling strategy
- Test strategy
- File organization

---

## Global Patterns

### Validation

```tsx
// Always validate inputs with Zod
const InputSchema = z.object({
  prompt: z.string().min(1).max(10000),
  // ...
});

const validated = InputSchema.parse(input);
```

### Error Handling

```tsx
// Never silent failures
try {
  // operation
} catch (error) {
  throw new Error(`Operation failed: ${error.message}`);
}
```

### Testing

```tsx
// Co-located tests
// src/module-name.ts
// src/module-name.test.ts (same directory)
```

---

**Total Pages Needed:** One detailed page per module (available on request)  

**Purpose of This Page:** Quick reference for all modules with patterns and strategies

[Link to detailed module specs coming]

---