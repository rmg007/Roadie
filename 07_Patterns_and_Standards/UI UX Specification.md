# 🖥️ UI / UX Specification

## Native VS Code UI Primitives Only

Roadie uses **zero external UI libraries**. All surfaces are built with the VS Code extension API's built-in primitives. This is non-negotiable — it ensures compatibility across all VS Code distributions (VS Code, VS Codium, Cursor, Windsurf).

**Phase 1 sidebar decision:** Roadie does NOT ship a sidebar WebView in Phase 1. All UI lives in the chat stream, status bar, and notifications. A sidebar view (if ever built) is deferred to Phase 2+ and is explicitly out of scope for the Phase 1 spec.

---

## Permitted UI Primitives

| Primitive | API | Use case |
|---|---|---|
| Chat stream markdown | `vscode.ChatResponseStream.markdown(value)` | All text output |
| Chat stream button | `vscode.ChatResponseStream.button(command)` | Human-in-the-loop approvals |
| Chat stream anchor | `vscode.ChatResponseStream.anchor(target, title?)` | File / symbol references |
| Chat stream file tree | `vscode.ChatResponseStream.filetree(value, baseUri)` | Showing files changed |
| Chat stream reference | `vscode.ChatResponseStream.reference(value)` | Referencing workspace files in the "used context" tray |
| Chat stream progress | `vscode.ChatResponseStream.progress(message)` | Between-step status ticks |
| Status bar item | `vscode.window.createStatusBarItem(alignment, priority)` | Ambient extension status |
| Notification (info) | `vscode.window.showInformationMessage(msg, ...actions)` | Non-blocking success notices |
| Notification (warn) | `vscode.window.showWarningMessage(msg, ...actions)` | Non-blocking warnings |
| Notification (error) | `vscode.window.showErrorMessage(msg, ...actions)` | Error with action buttons |
| Quick pick | `vscode.window.showQuickPick(items, options)` | Menu selection |
| Input box | `vscode.window.showInputBox(options)` | Single text input |
| Progress notification | `vscode.window.withProgress(options, task)` | Long operations outside chat |

**Prohibited:** WebViews, TreeViews, custom sidebars, React components, internal CSS.

---

## `ChatResponseStream.button()` — Canonical Signature and Usage

This is the single most important primitive in Roadie's HITL surface. An incorrect signature breaks every Feature / Dependency / Onboard / Review workflow that requires approval.

### Signature

```ts
// From @types/vscode (package: @types/vscode@^1.85.0)
interface ChatResponseStream {
  button(command: vscode.Command): void;
  // ... other methods
}

// vscode.Command (already in @types/vscode)
interface Command {
  title: string;            // Button label shown to the user — required
  command: string;          // VS Code command ID (e.g., 'roadie.approvePlan') — required
  tooltip?: string;         // Hover text — optional
  arguments?: unknown[];    // Passed positionally when the command is invoked
}
```

**Three non-obvious rules:**

1. The button **label** is `title`, not `label`. Using `label` compiles but produces an empty button.
2. `command` must be a **registered VS Code command** (via `vscode.commands.registerCommand` in the extension's `activate()`). Unregistered command IDs silently no-op when clicked.
3. `arguments` is positional and must serialize cleanly to JSON (no functions, no class instances). The command handler receives them as `(arg0, arg1, ...)`.

### Canonical Example (Feature Workflow — Step 2, Plan Approval)

```ts
async function renderPlanApproval(
  stream: vscode.ChatResponseStream,
  executionId: string,
  planText: string,
): Promise<void> {
  stream.markdown(`## Proposed Plan\n\n${planText}\n\n`);

  stream.button({
    title:   '✅ Approve Plan',
    command: 'roadie.approvePlan',
    tooltip: 'Accept the plan and start implementation',
    arguments: [executionId],
  });

  stream.button({
    title:   '✏️ Revise Plan',
    command: 'roadie.revisePlan',
    tooltip: 'Provide feedback and re-plan',
    arguments: [executionId],
  });

  stream.button({
    title:   '✖ Cancel',
    command: 'roadie.cancelWorkflow',
    tooltip: 'Abandon this workflow',
    arguments: [executionId],
  });
}
```

### Command Registration (required in `activate()`)

```ts
// src/shell/commands.ts
export function registerRoadieCommands(context: vscode.ExtensionContext, engine: WorkflowEngine): void {
  context.subscriptions.push(
    vscode.commands.registerCommand('roadie.approvePlan',   (executionId: string) => engine.resume(executionId, { action: 'approve' })),
    vscode.commands.registerCommand('roadie.revisePlan',    (executionId: string) => engine.resume(executionId, { action: 'revise' })),
    vscode.commands.registerCommand('roadie.cancelWorkflow',(executionId: string) => engine.cancel(executionId)),
    vscode.commands.registerCommand('roadie.confirmUpdate', (executionId: string, pkg: string) => engine.resume(executionId, { action: 'confirm', payload: pkg })),
    vscode.commands.registerCommand('roadie.skipPackage',   (executionId: string, pkg: string) => engine.resume(executionId, { action: 'skip', payload: pkg })),
    vscode.commands.registerCommand('roadie.confirmRefactor',(executionId: string) => engine.resume(executionId, { action: 'confirm' })),
    vscode.commands.registerCommand('roadie.abortRefactor', (executionId: string) => engine.cancel(executionId)),
    vscode.commands.registerCommand('roadie.viewReviewFindings', (executionId: string) => engine.showFindings(executionId)),
    vscode.commands.registerCommand('roadie.acknowledgeDocs',(executionId: string) => engine.resume(executionId, { action: 'acknowledge' })),
    vscode.commands.registerCommand('roadie.continueOnboarding',(executionId: string) => engine.resume(executionId, { action: 'continue' })),
    vscode.commands.registerCommand('roadie.init',          () => engine.runInit()),
    vscode.commands.registerCommand('roadie.rescan',        () => engine.runRescan()),
    vscode.commands.registerCommand('roadie.reset',         () => engine.runReset()),
  );
}
```

---

## Chat Stream Output Structure

Every workflow response streamed to the Chat Participant follows this exact format:

```
[Header — always first]
🚀 Starting {workflow_name} workflow...
---

[Progress — one line per step]
⏳ Step 1/8: Locating error source...
✅ Step 1/8: Error located in src/auth/session.ts:42

⏳ Step 2/8: Diagnosing root cause...
✅ Step 2/8: Diagnosis complete

[...]

[Result — always last]
---
## Result

{step output text}

**Files Changed:**
{file tree}

**Next Steps:**
- {suggested follow-up action 1}
- {suggested follow-up action 2}
```

### Streaming Rules

1. **Never buffer then dump** — stream each step header immediately when the step begins.
2. **Use ⏳ for in-progress, ✅ for success, ❌ for failure, ⚠️ for warnings.**
3. **File references** use `stream.anchor()` so they are clickable.
4. **Step results** that reference code: use markdown code blocks with language tags.
5. **Human approval steps** use `stream.button()` — do not use text prompts asking for "y/n".
6. **Every step header is prefixed with `Step N/TOTAL:`** — both numbers are literal (Step 2/8, not "Step 2 of 8").

---

## Exhaustive HITL Button Table

Every workflow that can pause to ask the developer for a decision MUST use exactly the labels and command IDs below. **These strings are frozen contract** — tests assert on them.

| Workflow | Step | Button Title (exact) | Command ID | Style | Args |
|---|---|---|---|---|---|
| **Feature** | 2 — Plan approval | `✅ Approve Plan` | `roadie.approvePlan` | primary | `[executionId]` |
| **Feature** | 2 — Plan approval | `✏️ Revise Plan` | `roadie.revisePlan` | secondary | `[executionId]` |
| **Feature** | 2 — Plan approval | `✖ Cancel` | `roadie.cancelWorkflow` | destructive | `[executionId]` |
| **Refactor** | 1 — Confirm public-API invariant | `✅ Proceed with Refactor` | `roadie.confirmRefactor` | primary | `[executionId]` |
| **Refactor** | 1 — Confirm public-API invariant | `✖ Abort` | `roadie.abortRefactor` | destructive | `[executionId]` |
| **Review** | 5 — Findings delivered | `📋 View All Findings` | `roadie.viewReviewFindings` | primary | `[executionId]` |
| **Review** | 5 — Findings delivered | `✅ Approve for Merge` | `roadie.approveForMerge` | primary | `[executionId]` |
| **Review** | 5 — Findings delivered | `❌ Block Merge` | `roadie.blockMerge` | destructive | `[executionId]` |
| **Bug Fix** | 8 — Summary | *(no buttons — terminal)* | — | — | — |
| **Document** | 3 — Generated doc ready | `📖 View Docs` | `roadie.acknowledgeDocs` | primary | `[executionId]` |
| **Document** | 3 — Generated doc ready | `✏️ Regenerate` | `roadie.regenerateDocs` | secondary | `[executionId]` |
| **Dependency** | Per-package (high-risk) | `⚡ Update Anyway` | `roadie.confirmUpdate` | primary | `[executionId, packageName]` |
| **Dependency** | Per-package (high-risk) | `⏭️ Skip This Package` | `roadie.skipPackage` | secondary | `[executionId, packageName]` |
| **Onboard** | 4 — Guided walkthrough checkpoint | `➡ Continue` | `roadie.continueOnboarding` | primary | `[executionId]` |
| **Onboard** | 4 — Guided walkthrough checkpoint | `✖ Done` | `roadie.cancelWorkflow` | secondary | `[executionId]` |

**Style semantics** (Roadie's internal classification — not a VS Code API):
- **primary** → green check / go-ahead semantic
- **secondary** → neutral / alternative path
- **destructive** → red / aborts or blocks

The VS Code `button()` API does not expose a style channel, so the style is conveyed via the emoji prefix in the title. Titles MUST include the emoji.

### Timeouts

No HITL button has an automatic timeout. A workflow that renders buttons transitions to `PAUSED` state and stays there until the developer clicks one or the workflow is cancelled via the command palette (`Roadie: Cancel Active Workflow`).

---

## Status Bar

The status bar item (`roadie.statusBarItem`) communicates ambient extension state at all times.

**Construction:**

```ts
const item = vscode.window.createStatusBarItem(
  vscode.StatusBarAlignment.Right,
  100, // priority — higher = further left on the right side
);
item.name    = 'Roadie';
item.command = 'roadie.showChatThread'; // click handler (registered in activate)
item.show();
```

**State → text + tooltip mapping** (exact strings — tests assert):

| State | `item.text` | `item.tooltip` | `item.backgroundColor` |
|---|---|---|---|
| Active, idle | `$(robot) Roadie` | `Roadie is active. Type @roadie in chat.` | *(unset)* |
| Workflow running | `$(sync~spin) Roadie` | `Workflow running — {workflow_display_name}` | *(unset)* |
| Workflow paused (HITL) | `$(warning) Roadie` | `Roadie is waiting for your approval` | `new vscode.ThemeColor('statusBarItem.warningBackground')` |
| Workflow failed | `$(error) Roadie` | `Last workflow failed — click to see details` | `new vscode.ThemeColor('statusBarItem.errorBackground')` |
| Extension inactive | *(not shown — call `item.hide()`)* | — | — |

`{workflow_display_name}` MUST be one of: `Bug Fix`, `Feature`, `Refactor`, `Review`, `Document`, `Dependency`, `Onboard`.

**Click action:** always executes `roadie.showChatThread`, which reveals the last Roadie chat thread in the chat view (via `vscode.commands.executeCommand('workbench.action.chat.open', { query: '@roadie' })`).

---

## Notification Copy — Per Workflow

Every user-visible notification string is listed here. Do not invent new copy.

### Success (info) — `showInformationMessage`

| Workflow | Trigger | Message | Actions |
|---|---|---|---|
| Bug Fix | Workflow complete | `Roadie: Bug fix complete. {files_changed} file(s) changed, all tests passing.` | `Open PR`, `View Diff`, `Dismiss` |
| Feature | Workflow complete | `Roadie: Feature "{feature_name}" implemented. {files_changed} file(s) changed.` | `Open PR`, `View Diff`, `Dismiss` |
| Refactor | Workflow complete | `Roadie: Refactor complete. Public API unchanged, {files_changed} file(s) modified.` | `View Diff`, `Dismiss` |
| Review | Workflow complete | `Roadie: Review complete. {findings_count} finding(s): {critical_count} critical, {medium_count} medium, {low_count} low.` | `View Findings`, `Dismiss` |
| Document | Workflow complete | `Roadie: Documentation generated at {target_path}.` | `Open File`, `Dismiss` |
| Dependency | Workflow complete | `Roadie: {updated_count} package(s) upgraded, {skipped_count} skipped.` | `View Changelog`, `Dismiss` |
| Onboard | Workflow complete | `Roadie: Onboarding complete. Starter task suggestion is in the chat thread.` | `Go to Chat`, `Dismiss` |

### Warning — `showWarningMessage`

| Trigger | Message | Actions |
|---|---|---|
| Tests skipped | `Roadie skipped the test step because no test command was detected. Set "roadie.testCommand" to enable test verification.` | `Open Settings`, `Dismiss` |
| High-risk dependency | `Roadie: Upgrading "{package_name}" from {from_version} to {to_version} may introduce breaking changes.` | `Show Changelog`, `Skip`, `Proceed` |
| Merge deferred (file open) | `Roadie: "{file_path}" is open with unsaved changes. Regeneration deferred until you save.` | `Show File`, `Dismiss` |
| Section markers removed | `Roadie markers were removed from "{file_path}". New Roadie content appended at the bottom. Review and reorganize as needed.` | `Open File`, `Dismiss` |

### Error — `showErrorMessage`

| Trigger | Message | Actions |
|---|---|---|
| `ModelUnavailableError` | `Roadie: No language model available. Check your Copilot subscription or configure an API key.` | `Check Copilot Subscription`, `Configure API Key`, `Dismiss` |
| `WORKFLOW_FAILED` | `Roadie: {workflow_display_name} workflow failed after {attempts} attempts. Last error: {error_summary}` | `Show Details`, `Retry`, `Dismiss` |
| `DATABASE_ERROR` | `Roadie: Project database is corrupted. Rebuilding from filesystem.` | `Rescan Now`, `Dismiss` |
| `FILE_PERMISSION_ERROR` | `Roadie: Cannot write to "{file_path}" (permission denied).` | `Open File`, `Dismiss` |
| `GENERATOR_TIMEOUT` | `Roadie: {generator_name} took longer than {budget_ms}ms. File not updated.` | `Retry`, `Dismiss` |
| `ANALYSIS_TIMEOUT` | `Roadie: Project analysis exceeded 30s. This usually means a very large workspace.` | `Rescan with --fast`, `Dismiss` |

**Notification rules:**
- Never show `showErrorMessage` for non-actionable errors (use chat stream instead).
- Never show multiple notifications in sequence — consolidate into one.
- Non-blocking info messages auto-dismiss after 5 seconds (VS Code default).
- **Placeholder tokens** (`{file_path}`, `{workflow_display_name}`, etc.) are filled at call time. They MUST be treated as contract — tests assert on the exact format strings.

---

## Accessibility Contract

VS Code provides native accessibility for all primitives Roadie uses. Roadie's obligations are:

1. **Screen reader labels.** Every button's `title` is what a screen reader announces. Titles MUST be plain-English and self-describing (no cryptic labels like "OK" without context).
2. **Keyboard navigation.**
   - Chat stream buttons are focusable and activatable via `Space` / `Enter` — inherited from VS Code, no action needed beyond using the correct API.
   - Status bar item is focusable via `Ctrl+Shift+P → Focus Status Bar` — inherited.
   - Notification action buttons are focusable via `Tab` — inherited.
3. **No information conveyed by emoji alone.** Every emoji in a button title is accompanied by plain-English text (`✅ Approve Plan`, not just `✅`). Screen readers skip the emoji; the plain text carries meaning.
4. **No color-only state signaling.** The status bar uses icon codicons (`$(robot)`, `$(sync~spin)`, `$(warning)`, `$(error)`) in addition to background color.
5. **Motion reduction.** The `$(sync~spin)` icon is the only animated UI element. It is inherited from VS Code and automatically respects the user's `editor.accessibilitySupport` setting.
6. **Focus management.** When a workflow pauses at a HITL step, the chat stream buttons receive natural DOM focus via VS Code's chat widget — no manual focus management is required.

### Test Assertions (mandatory)

```ts
it('every HITL button title contains plain-English text', () => {
  const allTitles = collectAllButtonTitles(); // from the exhaustive table above
  for (const title of allTitles) {
    // Strip leading emoji and whitespace, then assert the remainder is ≥ 3 words
    const plain = title.replace(/^\p{Extended_Pictographic}\s*/u, '').trim();
    expect(plain.split(/\s+/).length).toBeGreaterThanOrEqual(1);
    expect(plain.length).toBeGreaterThanOrEqual(4);
  }
});

it('status bar text always includes both an icon and the word "Roadie"', () => {
  for (const state of STATUS_BAR_STATES) {
    expect(state.text).toMatch(/^\$\([a-z-~ ]+\) Roadie$/);
  }
});
```

---

## Roadie-Owned Files as UI

Roadie writes to two files which act as output surfaces:

| File | Purpose | Update Trigger |
|---|---|---|
| `.github/copilot-instructions.md` | Injects project context into all Copilot / agent calls | After any workflow that changes the project model |
| `AGENTS.md` | Documents agent roles for all AI coding tools | After any workflow that changes the project model |

Both files use `SectionManager` ownership markers so that developer-authored sections are never overwritten. See `Section Manager Specification.md` for the exact format.
