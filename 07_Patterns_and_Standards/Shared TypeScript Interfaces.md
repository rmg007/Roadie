# 🔧 Shared TypeScript Interfaces

## Single Source of Truth for All Module Contracts

Every module in Phase 1 exports/imports types defined here. If a module needs a type that doesn't exist on this page, it's a spec bug.

**Key Principle:** Types flow outward from this file. External modules import from `types.ts`. No circular imports.

---

## Intent Classification Types

```tsx
/**
 * The 8 intent types that the classifier can produce.
 * Each maps to a workflow or to passthrough enrichment.
 */
type IntentType =
  | 'bug_fix'
  | 'feature'
  | 'refactor'
  | 'review'
  | 'document'
  | 'dependency'
  | 'onboard'
  | 'general_chat';

/**
 * Result of intent classification.
 * Produced by IntentClassifier, consumed by ChatParticipantHandler.
 * 
 * @property intent The classified intent type
 * @property confidence Score 0.0-1.0 indicating classifier certainty
 * @property signals List of keywords/patterns matched (for debugging)
 * @property requiresLLM True if local classifier confidence < 0.7, indicating LLM fallback needed
 */
interface ClassificationResult {
  intent: IntentType;
  confidence: number; // 0.0 to 1.0
  signals: string[]; // Matched keywords, e.g. ["fix", "error", "login"]
  requiresLLM: boolean;
}
```

## Intent Classifier Interface

```tsx
/**
 * Two-tier intent classifier.
 * Tier 1: Local keyword/regex (instant, zero cost).
 * Tier 2: LLM classification piggybacked on the first response (double-duty call).
 *
 * The ChatParticipantHandler uses this interface:
 * 1. Call classify(prompt) for local classification
 * 2. If requiresLLM is true, prepend getClassificationPromptPrefix() to the system prompt
 * 3. After receiving the LLM response, call parseClassification(responseText)
 * 4. If parseClassification returns null, fall back to general_chat
 */
interface IntentClassifier {
  /** Local classification (instant, zero cost) */
  classify(prompt: string): ClassificationResult;
  /** Parse LLM classification from a response that includes structured output */
  parseClassification(responseText: string): ClassificationResult | null;
  /** Generate the structured-output prefix to prepend to the system prompt */
  getClassificationPromptPrefix(): string;
}
```

---

## Workflow Engine Types

```tsx
/**
 * Workflow execution states.
 * Transitions: PENDING → RUNNING → [WAITING_PARALLEL | RETRYING] → COMPLETED | PAUSED | FAILED | CANCELLED
 */
enum WorkflowState {
  PENDING = 'PENDING', // Created, not yet started
  RUNNING = 'RUNNING', // Currently executing a step
  WAITING_PARALLEL = 'WAITING_PARALLEL', // Waiting for parallel branches
  RETRYING = 'RETRYING', // Step failed, retrying with escalation
  PAUSED = 'PAUSED', // Step failed 3x, awaiting developer intervention
  COMPLETED = 'COMPLETED', // All steps succeeded
  FAILED = 'FAILED', // Workflow aborted (only on cancellation)
  CANCELLED = 'CANCELLED', // Developer cancelled
}

/**
 * Declarative definition of a workflow.
 * The workflow engine interprets this and executes it.
 */
interface WorkflowDefinition {
  /** Unique ID for this workflow: 'bug_fix', 'feature', 'refactor', etc. */
  id: string;
  /** Human-readable name for logging/UI */
  name: string;
  /** Sequential list of steps to execute */
  steps: WorkflowStep[];
  /** Optional: hook called after all steps complete */
  onComplete?: (results: StepResult[]) => Promise<WorkflowResult>;
}

/**
 * Single step in a workflow.
 */
interface WorkflowStep {
  /** Unique ID within the workflow */
  id: string;
  /** Human-readable name for progress updates */
  name: string;
  /** 'sequential': execute after previous step completes.
   *  'parallel': execute concurrently with siblings (via Promise.allSettled()).
   *  'conditional': execute based on predicate from previous step result. */
  type: 'sequential' | 'parallel' | 'conditional';
  /** The role the subagent plays (diagnostician, fixer, planner, etc.) */
  agentRole: AgentRole;
  /** Starting model tier for this step (free/standard/premium) */
  modelTier: ModelTier;
  /** Which tools the subagent can invoke */
  toolScope: ToolScope;
  /** System prompt template (may contain {variable} placeholders) */
  promptTemplate: string;
  /** Maximum time to wait for step to complete (milliseconds) */
  timeoutMs: number;
  /** Maximum retries with escalation before reporting failure */
  maxRetries: number;
  /** For parallel steps: list of sub-steps to run concurrently */
  branches?: WorkflowStep[];
  /** For conditional steps: predicate that returns next step ID or null */
  condition?: (previousResult: StepResult) => string | null;
}

/**
 * Context passed to workflow engine and threaded through each step.
 */
interface WorkflowContext {
  /** The developer's original prompt */
  prompt: string;
  /** Classification result from IntentClassifier */
  intent: ClassificationResult;
  /** Project model (for context injection) */
  projectModel: ProjectModel;
  /** Stream for sending progress updates to chat UI */
  chatResponseStream: vscode.ChatResponseStream;
  /** Token for cancelling the workflow */
  cancellationToken: vscode.CancellationToken;
  /** Results from previous step (if any) */
  previousStepResults?: StepResult[];
}

/**
 * Result of executing a single workflow step.
 */
interface StepResult {
  /** Step ID */
  stepId: string;
  /** Success, failure, skipped, or cancelled */
  status: 'success' | 'failed' | 'skipped' | 'cancelled';
  /** Text output from the subagent */
  output: string;
  /** Results from any tools the subagent invoked */
  toolResults?: ToolCallResult[];
  /** Token usage for this step */
  tokenUsage: { input: number; output: number };
  /** Number of attempts before success (or max attempts if failed) */
  attempts: number;
  /** Which model was used for the successful attempt (e.g., 'gpt-4.1') */
  modelUsed: string;
  /** Human-readable error message if status === 'failed' */
  error?: string;
}

/**
 * Final result of a complete workflow execution.
 */
interface WorkflowResult {
  /** Workflow ID */
  workflowId: string;
  /** Final state */
  state: WorkflowState;
  /** Results from all steps */
  stepResults: StepResult[];
  /** Total execution time in milliseconds */
  duration: number;
  /** Which model tiers were used (for cost tracking) */
  modelTiersUsed: ModelTier[];
  /** Human-readable summary for chat display */
  summary: string;
}
```

---

## Agent Spawner Types

```tsx
/**
 * Role-specific prompt configurations (not separate Chat Participants).
 * Each role has its own system prompt, tool scope, and model preference.
 */
type AgentRole =
  | 'diagnostician' // Locate and diagnose errors
  | 'fixer' // Generate and apply fixes
  | 'planner' // Plan feature implementation
  | 'database_agent' // Schema changes, migrations, queries
  | 'backend_agent' // API endpoints, business logic
  | 'frontend_agent' // UI components, state, styling
  | 'refactorer' // Incremental code restructuring
  | 'security_reviewer' // Security analysis (OWASP, injection, secrets)
  | 'performance_reviewer' // Performance analysis (N+1, memory leaks)
  | 'quality_reviewer' // Code quality (duplication, naming, patterns)
  | 'test_reviewer' // Test coverage and edge cases
  | 'standards_reviewer' // Project convention compliance
  | 'documentarian' // Generate and update documentation
  | 'project_analyzer'; // Build and maintain project model

/**
 * Cost tier for LLM calls.
 */
type ModelTier = 'free' | 'standard' | 'premium';

/**
 * Tool scoping per step type.
 */
type ToolScope = 'research' | 'implementation' | 'review' | 'documentation';

/**
 * Configuration passed to AgentSpawner to create a subagent.
 */
interface AgentConfig {
  /** Agent role (determines system prompt) */
  role: AgentRole;
  /** Starting model tier */
  modelTier: ModelTier;
  /** Tool scope (research/implementation/review/documentation) */
  tools: ToolScope;
  /** System prompt template (may contain {variable} placeholders) */
  promptTemplate: string;
  /** Context injected into prompt (tech stack, patterns, etc.) */
  context: Record<string, unknown>;
  /** Maximum time to wait for agent response (milliseconds) */
  timeoutMs: number;
}

/**
 * Result from spawning and executing an agent.
 */
interface AgentResult {
  /** Text output from the agent */
  output: string;
  /** Results from any tool calls */
  toolResults: ToolCallResult[];
  /** Token usage */
  tokenUsage: { input: number; output: number };
  /** 'success', 'failed', or 'timeout' */
  status: 'success' | 'failed' | 'timeout';
  /** Which model was used (e.g., 'gpt-4.1', 'claude-sonnet-4.6') */
  model: string;
  /** Error message if status !== 'success' */
  error?: string;
}

/**
 * Result from a tool invocation within an agent.
 */
interface ToolCallResult {
  /** Name of the tool called (e.g., 'readFile', 'searchWorkspace') */
  tool: string;
  /** Input to the tool */
  input: unknown;
  /** Output from the tool */
  output: unknown;
  /** Whether the tool call succeeded */
  success: boolean;
  /** Error message if success === false */
  error?: string;
}
```

---

## Project Model Types

```tsx
/**
 * Main interface for the project model.
 * Provides typed access to all project information.
 */
interface ProjectModel {
  /** Tech stack of the project (languages, frameworks, versions) */
  getTechStack(): TechStackEntry[];
  /** Directory structure and roles (source, test, config) */
  getDirectoryStructure(): DirectoryNode;
  /** Detected coding patterns (export style, test conventions, etc.) */
  getPatterns(): DetectedPattern[];
  /** Developer preferences from configuration */
  getPreferences(): DeveloperPreferences;
  /** Build, test, dev, and lint commands */
  getCommands(): ProjectCommand[];
  /** Serialized context string for LLM prompts.
   *  Accepts optional parameters for token budgeting and scope filtering.
   *  @param options.maxTokens - Token budget for the serialized output
   *  @param options.scope - Filter to specific context categories
   *  @param options.relevantPaths - Only include context relevant to these paths
   */
  toContext(options?: {
    maxTokens?: number;
    scope?: 'full' | 'stack' | 'structure' | 'commands' | 'patterns';
    relevantPaths?: string[];
  }): ProjectContext;
  /** Apply a delta update to the model */
  update(delta: ProjectModelDelta): void;
}

/**
 * Single tech stack entry (language, framework, runtime, etc.).
 */
interface TechStackEntry {
  /** 'language', 'framework', 'runtime', 'orm', 'test_tool', 'build_tool', 'package_manager' */
  category: string;
  /** Name of the tech (TypeScript, React, Vitest) */
  name: string;
  /** Semver version if detectable */
  version?: string;
  /** File where this was detected (package.json, tsconfig.json) */
  sourceFile: string;
}

/**
 * Represents the directory tree structure.
 */
interface DirectoryNode {
  /** Absolute path */
  path: string;
  /** 'directory' or 'file' */
  type: 'directory' | 'file';
  /** 'source', 'test', 'config', 'output', 'static' (if applicable) */
  role?: string;
  /** Detected language for code files (typescript, javascript, python, etc.) */
  language?: string;
  /** Child nodes if type === 'directory' */
  children?: DirectoryNode[];
}

/**
 * A detected coding pattern or convention.
 */
interface DetectedPattern {
  /** Category: 'export_style', 'test_convention', 'error_handling', etc. */
  category: string;
  /** Human-readable description (e.g., "Uses named exports only") */
  description: string;
  /** Evidence: files sampled, match count, confidence */
  evidence: {
    files: string[];
    matchCount: number;
    confidence: number;
  };
  /** Overall confidence 0.0-1.0 */
  confidence: number;
}

/**
 * A command detected in the project (build, test, dev, lint, format).
 */
interface ProjectCommand {
  /** Name: 'build', 'test', 'dev', 'lint', 'format' */
  name: string;
  /** Full command string (e.g., 'npm run test') */
  command: string;
  /** Where it came from (package.json, Makefile, etc.) */
  sourceFile: string;
  /** Type of command */
  type: 'build' | 'test' | 'dev' | 'lint' | 'format' | 'other';
}

/**
 * Developer preferences (from configuration and detected patterns).
 */
interface DeveloperPreferences {
  testCommand?: string; // Custom test runner override
  modelPreference?: 'economy' | 'balanced' | 'quality';
  telemetryEnabled: boolean;
  autoCommit: boolean;
}

/**
 * Serialized context for LLM prompts.
 * Produced by ProjectModel.toContext().
 */
interface ProjectContext {
  techStack: TechStackEntry[];
  directoryStructure: DirectoryNode;
  patterns: DetectedPattern[];
  commands: ProjectCommand[];
  /** The full serialized string ready for prompt injection */
  serialized: string;
}

/**
 * Delta update for partial model changes.
 */
interface ProjectModelDelta {
  techStack?: TechStackEntry[];
  directories?: DirectoryNode[];
  patterns?: DetectedPattern[];
  commands?: ProjectCommand[];
}
```

---

## File Generation Types

```tsx
/**
 * Types of files Roadie generates in Phase 1.
 */
type GeneratedFileType =
  | 'copilot_instructions' // .github/copilot-instructions.md
  | 'agents_md'; // AGENTS.md at project root

/**
 * Generated file with content and metadata.
 */
interface GeneratedFile {
  /** File type */
  type: GeneratedFileType;
  /** Full path relative to workspace root */
  path: string;
  /** File content (markdown) */
  content: string;
  /** SHA-256 hash of content for change detection */
  contentHash: string;
  /** Whether the file was actually written (false if identical to existing) */
  written: boolean;
}
```

---

## Error Types

```tsx
/**
 * Standard error shape thrown by Roadie modules.
 */
interface RoadieError extends Error {
  /** Error code for programmatic handling */
  code: string;
  /** 'validation' | 'timeout' | 'cancelled' | 'escalation' | 'external' */
  category: string;
  /** Whether the error should be shown to the developer */
  userFacing: boolean;
  /** Detailed context for debugging */
  context?: Record<string, unknown>;
}

/**
 * Specific error for step execution failures.
 */
class StepExecutionError extends Error {
  constructor(
    public stepId: string,
    public attempt: number,
    public maxRetries: number,
    message: string
  ) {
    super(message);
  }
}
```

---

## VS Code API Wrappers

```tsx
/**
 * Wrapper for VS Code's Language Model API.
 * Returned by vscode.lm.selectChatModels().
 */
type LanguageModelChat = ReturnType<typeof vscode.lm.selectChatModels>[0];

/**
 * Wrapper for VS Code's chat response stream.
 */
type ChatResponseStream = vscode.ChatResponseStream;
```

---

## Summary

- **Import Path:** `src/types.ts`
- **Usage:** Every module imports types it needs from here
- **Add New Type:** Only add types here if they cross module boundaries. Internal types belong in their module.
- **Breaking Changes:** Changing a type breaks all modules that use it. Careful!

**Next:** Go to Module Build Order to see the exact sequence for building modules.