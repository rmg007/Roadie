# Developer Environment Checks

## VS Code Settings and Extension Checks for the Recommendation Engine

**Phase:** 2 (MCP Server — part of `get_recommendations` tool)
**Module:** `src/recommendations/environment-checks.ts`
**Dependencies:** VsCodeShellProvider (for `vscode.extensions.all` and `vscode.workspace.getConfiguration()`)

---

## Purpose

When `roadie/get_recommendations` is called, Roadie inspects the developer's VS Code environment for suboptimal configurations that degrade Copilot performance. These checks are passive (read-only), require no user configuration, and produce actionable recommendations.

---

## Five Checks

| # | Check | Setting/API | Priority | Trigger |
|---|-------|-------------|----------|---------|
| 1 | Conflicting AI extensions | `vscode.extensions.all` | high | TabNine, Kite, Cody, Codeium, or Amazon Q installed |
| 2 | Formatter on-type | `editor.formatOnType` | medium | Set to `true` |
| 3 | Local indexing disabled | `github.copilot.chat.advanced.workspace.codeSearchExternalIngest.enabled` | medium | Set to `false` or not set |
| 4 | NES disabled | `github.copilot.nextEditSuggestions.enabled` | low | Set to `false` or not set |
| 5 | No inline suggest debounce | `editor.inlineSuggest.minShowDelay` | low | Value < 200ms |

---

## Conflicting Extension IDs

These extension IDs are checked against `vscode.extensions.all`:

```typescript
const CONFLICTING_EXTENSIONS = [
  'tabnine.tabnine-vscode',           // TabNine
  'kiteco.kite',                       // Kite (discontinued but may still be installed)
  'sourcegraph.cody-ai',              // Sourcegraph Cody
  'codeium.codeium',                  // Codeium
  'amazonwebservices.aws-toolkit-vscode', // Amazon Q / CodeWhisperer
] as const;
```

**Why these specifically:** Each provides ghost-text (inline suggestions) that compete with Copilot for the same UI surface. Running two ghost-text providers creates race conditions in the editor widget, causing flicker, CPU spikes, and dropped suggestions.

**Excluded from the list:** Extensions that complement Copilot without conflicting (e.g., GitLens, ESLint, Prettier). Only extensions that provide competing inline completions are flagged.

---

## Standalone Mode Behavior

In standalone MCP mode (`npx roadie-mcp --project .`), the VS Code APIs are not available. The `getEnvironmentRecommendations` function detects this via the provider pattern:

```typescript
export async function getEnvironmentRecommendations(
  provider: ShellProvider
): Promise<Recommendation[]> {
  // Environment checks require VS Code APIs
  if (!(provider instanceof VsCodeShellProvider)) {
    return []; // Silently skip — no VS Code to inspect
  }
  // ... run checks
}
```

This follows the existing core/shell split pattern from Phase 2 architecture.

---

## Integration Point

The recommendation handler in `src/mcp/tools/get-recommendations.ts` calls environment checks after existing checks:

```typescript
async function handleGetRecommendations(input: {}, provider: ShellProvider): Promise<RecommendationsOutput> {
  const recommendations: Recommendation[] = [];

  // Existing checks
  recommendations.push(...await getConfigRecommendations(projectModel));
  recommendations.push(...await getPatternRecommendations(projectModel));
  recommendations.push(...await getFeatureRecommendations(config));

  // Environment checks (Phase 2 addition)
  recommendations.push(...await getEnvironmentRecommendations(provider));

  // Sort by priority: high > medium > low
  recommendations.sort((a, b) => PRIORITY_ORDER[a.priority] - PRIORITY_ORDER[b.priority]);

  return { recommendations };
}
```

---

## Performance

All five checks read in-memory VS Code state. No file system access, no network calls, no database queries. Total execution time: < 5ms.

---

## Testing

See test cases in `08_Integration_and_Testing/Testing, Error Handling & Security.md` under "Developer Environment Recommendation Tests".

Mock strategy: Use `vi.mock('vscode')` to provide controlled `extensions.all` arrays and `workspace.getConfiguration()` return values. Each check is independently testable.

---

## Future Extensions (Phase 2.5+)

When adaptive learning is enabled, environment checks could be prioritized based on observed impact:
- If the developer enables NES after seeing the recommendation, record that as a positive signal
- If the developer dismisses the same recommendation 3 times, suppress it permanently
- Track whether disabling conflicting extensions correlates with improved workflow success rates

These extensions depend on Phase 2.5 infrastructure and are NOT part of the initial implementation.
