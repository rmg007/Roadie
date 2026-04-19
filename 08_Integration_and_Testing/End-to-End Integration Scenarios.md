# 🌀 End-to-End Integration Scenarios

## 5 Complete Flows from User Prompt to Final Result

These scenarios show the complete data flow through Phase 1, including happy paths and failure recovery.

---

## Scenario 1: Bug Fix Workflow (Happy Path)

**User Prompt:** "The login page throws a 500 error after the last deploy"

### Step 1: Intent Classification

**Input:** Prompt string  

**Classifier:** Local tier  

**Signals:** "throws", "500 error", "login", "error", "after deploy"  

**Confidence:** 0.92 (multiple primary signals)  

**Result:** `intent='bug_fix', confidence=0.92, requiresLLM=false`  

**Latency:** 3ms

### Step 2: Chat Participant Routing

```tsx
// In chat-participant.ts handler
const classification = await classifier.classify(prompt);
if (classification.intent === 'bug_fix') {
  const context = new WorkflowContext({
    prompt,
    intent: classification,
    projectModel, // Rebuilt on first workflow of session
    chatResponseStream,
    cancellationToken,
  });
  
  const result = await workflowEngine.execute('bug_fix', context);
}
```

### Step 3: Workflow Engine Execution

Workflow state: `PENDING` → `RUNNING`

**Step 1: Locate Error Source**

- Agent role: `diagnostician`
- Model tier: free (GPT-4.1)
- Tools: file search, grep, git log
- Prompt template:
    
    ```
    You are a diagnostic agent. Locate the source of this error:
    {error_message}
    
    Project context:
    - Tech Stack: Next.js 14, Prisma, PostgreSQL
    - Test runner: Vitest
    - Directory: /src for source code
    
    Use file search and grep to locate the exact file and line. Return:
    - File path
    - Line number
    - Suspected cause
    ```
    
- Output:
    
    ```
    File: /src/api/auth/login.ts:42
    Error: Line 42 references `user.profile.id` without null check.
    Suspected cause: Migration removed profile auto-join in the recent deploy.
    ```
    
- Result: Success

**Step 2: Diagnose Root Cause**

- Agent role: `diagnostician`
- Model tier: **standard** (Claude Sonnet) — More nuanced
- Tools: code read, pattern search
- Uses Step 1 output as context
- Output:
    
    ```
    Root cause: Null reference exception
    Location: /src/api/auth/login.ts:42
    
    In the recent migration, the user-profile relationship changed
    from auto-joined to lazy-loaded. The code at line 42 assumes
    `user.profile` is always populated, but after the migration,
    it can be null.
    
    Fix approach: Add null check or explicitly load profile.
    ```
    
- Result: Success

**Step 3: Generate and Apply Fix**

- Agent role: `fixer`
- Model tier: free (GPT-4.1)
- Tools: code read, code edit, file write
- Prompt includes Step 2 diagnosis
- Output:
    
    ```tsx
    // /src/api/auth/login.ts at line 42
    
    // BEFORE:
    const profileId = user.profile.id;
    
    // AFTER:
    const profileId = user.profile?.id || user.id;
    ```
    
- Result: Success, file written

**Step 4: Verify Fix (Run Tests)**

- No agent; shell command
- Command: `npm run test` (detected from package.json)
- Output:
    
    ```
    PASS src/api/auth/login.test.ts
    PASS src/api/auth/profile.test.ts
    ...
    Tests: 42 passed, 0 failed
    ```
    
- Result: Success

**Steps 5-8:** (Scan for siblings, fix siblings, add regression test, summary)

- All succeed
- Regression test added to catch this pattern in future

### Step 4: Chat Response

```
Searching for error source...
✓ Located at /src/api/auth/login.ts:42

Diagnosing root cause...
✓ Null reference in user.profile access (migration issue)

Generating fix...
✓ Applied null-coalesce operator

Verifying with tests...
✓ 42 tests passed, 0 failed

Scanning for similar issues...
✓ Found 2 similar patterns in /src/api/profile/get.ts

Fixing similar issues...
✓ Fixed 2 additional instances

Adding regression test...
✓ New test: `should handle null profile gracefully`

**Summary:** Fixed null reference bug in login endpoint.
Root cause: Migration changed user-profile loading from auto-join to lazy-load.
Fix: Added null-coalesce operators to 3 locations.
All tests passing. Deploy safe.
```

### Workflow Result

```tsx
{
  workflowId: 'bug_fix',
  state: 'COMPLETED',
  duration: 45000, // 45 seconds
  modelTiersUsed: ['free', 'standard'],
  stepResults: [/*8 step results*/],
  summary: "Fixed null reference bug in login endpoint..."
}
```

---

## Scenario 2: Feature Development Workflow (with Revision)

**User Prompt:** "Add a dark mode toggle to the settings page"

### Step 1-2: Intent Classification → Chat Routing

Intent: `feature`, Confidence: 0.88  

Workflow: Feature Development

### Step 3: Analyze Requirements

Agent: `planner`, free tier  

Output: Structured requirements (UI, database flag, API endpoint, styling)

### Step 4: Present Plan for Approval

**Prompt:** Generate a feature plan with approval buttons  

**Chat Display:**

```
Analyzing requirements...
✓ Identified 4 components: database, backend, frontend, theme system

## Proposed Plan

1. **Database**: Add `settings.darkModeEnabled` boolean flag
2. **Backend**: GET/POST endpoints for user preference
3. **Frontend**: Toggle switch on settings page
4. **Styling**: CSS variables for light/dark theme

### Timeline
- Database: 5 min
- Backend: 10 min
- Frontend: 15 min

[Approve Plan] [Revise Plan]
```

**User Interaction:** Clicks "Revise Plan"  

Feedback: "Also detect system preference on first load"

### Step 5: Loop Back to Step 1

Analyze requirements again with feedback  

Re-present plan with system preference detection

**User Interaction:** Clicks "Approve Plan"

### Step 6: Delegate to Layer Agents (Parallel)

**Database Agent:**

 - free tier
- Generates Prisma schema change
- Output: Migration file + schema update

**Backend Agent:**

- free → standard tier (escalate if integration fails)
- Generates endpoints
- Output: GET `/api/settings/theme`, POST `/api/settings/theme`

**Frontend Agent:**

 - free tier
- Generates React component
- Output: `SettingsThemeToggle.tsx` with system preference detection

**Progress Display:**

```
Delegating to layer agents...
✓ Database schema updating...
✓ Backend endpoints generating...
✓ Frontend components scaffolding...
✓ All layers complete!
```

### Step 7-8: Integration → Tests → Quality Review → Summary

All succeed. Feature complete.

**Final Summary:**

```
Dark mode feature implemented.
- Database: Added darkModeEnabled flag to user settings
- Backend: GET/POST endpoints for theme preference, system detection
- Frontend: Toggle switch with system preference fallback
- Tests: 8 new tests, all passing
Ready to merge.
```

---

## Scenario 3: Code Review Workflow (5 Passes)

**User Prompt:** "Review my changes before I push"

**Git Diff:** Changes to authentication module (200 lines added/modified)

### Step 1: Parallel Review Passes

All 5 passes execute concurrently via Promise.allSettled():

**Pass 1: Security Review** (standard — critical)

- Checks OWASP Top 10, injection, auth flaws, secrets
- Output:
    
    ```
    Critical: No CSRF token on auth POST endpoints
    Warning: Verify JWT expiration is set
    Info: No hardcoded secrets detected
    ```
    

**Pass 2: Performance Review** (free)

- Checks N+1 queries, re-renders, memory leaks
- Output:
    
    ```
    Warning: UserService.getProfile() makes 2 queries without join
    ```
    

**Pass 3: Code Quality Review** (free)

- Naming, duplication, complexity
- Output:
    
    ```
    Suggestion: Function `validateAndSaveUser` does too much (>40 lines)
    ```
    

**Pass 4: Test Coverage Review** (free)

- Untested paths, edge cases
- Output:
    
    ```
    Warning: Missing test for invalid JWT case
    ```
    

**Pass 5: Standards Review** (free)

- Project conventions (from project model)
- Output:
    
    ```
    Info: Use named exports (project standard) — some functions use default export
    ```
    

### Step 2: Consolidation

**Chat Display:**

```
## Code Review: 5 Passes Completed

### 🔴 Critical Issues (1)
- [ ] CSRF protection missing on POST /auth/login

### 🟎┇ Warnings (3)
- [ ] N+1 query in UserService.getProfile()
- [ ] JWT expiration may not be set
- [ ] Untested path: invalid JWT handling

### 📑 Suggestions (2)
- [ ] Split `validateAndSaveUser` function (SRP)
- [ ] Use named exports consistently (project standard)

### Recommendation
Fix critical CSRF issue before merging. Address at least one warning.
Suggestions are improvements for future PRs.
```

---

## Scenario 4: Refactoring with Failure & Recovery

**User Prompt:** "Refactor the auth module—it's gotten complex"

### Workflow Execution

**Step 1: Analyze Current Structure** → Success  

**Step 2: Write Characterization Tests**

- Agent: `test_reviewer`, standard tier
- Generates tests that capture current behavior
- All existing auth functions tested
- **Result: 3 new tests generated**

**Step 3: Refactor Incrementally (Inner Loop)**

**Attempt 1:** Extract `validateCredentials` function

- Code edited
- Tests run: **FAIL** (one test broke)
- Change reverted
- Alternative approach: Extract into a class method instead

**Attempt 2:** Extract as method

- Code edited
- Tests run: **PASS**
- Continue to next refactoring

**Attempt 3:** Consolidate duplicate logic in `hashPassword`

- Code edited
- Tests run: **PASS**
- Continue

**Attempt 4:** Try to simplify promise chaining

- Code edited
- Tests run: **FAIL** (error handling broke)
- Change reverted
- Note: Public API invariant preserved (didn't change exports)

### Step 4: Summary

```
Refactoring complete (3/4 attempted improvements succeeded).

Before: 1 auth.ts file, 450 lines, high complexity
After: 3 files (auth.ts, validators.ts, crypto.ts), better separation

Changes preserved:
- All public exports unchanged
- All tests passing
- Behavior identical

Not refactored: Promise chaining simplification (risky, reverted)
```

---

## Scenario 5: Dependency Update with Escalation

**User Prompt:** "Update React to the latest version"

### Workflow Execution

**Step 1: Audit Dependency Tree** → Success  

Output: React 18.2.0 → 19.0.0 available

**Step 2: Identify CVEs** → Success  

Output: No known CVEs for v19.0.0

**Step 3: Check Breaking Changes** → Success (standard)

- Claude Sonnet analyzes changelog
- Output: 2 breaking changes detected
    - useEvent hook removed (we don't use it)
    - PropTypes removed (we don't use it)
    - No impact expected

**Step 4: Produce Upgrade Plan** → Success  

Output: Upgrade React 18.2 → 19.0 in package.json

**Step 5: Execute Upgrade & Test**

**Attempt 1:** Run `npm install`

- **FAILS:** Type error in `src/index.tsx:32` - React.FC signature changed
- Tries `npm run test` — many failures
- **Escalates to standard tier**

**Attempt 2:** standard tier agent analyzes React 19 migration guide

- Fixes: Update function component types from `React.FC<Props>` to plain functions
- Updates 8 components
- Runs tests: **PASS**

**Summary:**

```
React upgraded from 18.2.0 to 19.0.0
- 0 CVEs introduced
- 0 breaking changes affected our code
- 8 type annotations updated (React.FC deprecation)
- All tests passing
Ready to merge.
```

---

## Data Flow Diagram

```
User Prompt
    ↓
ChatParticipantHandler
    ↓
IntentClassifier (Local + optional LLM)
    ↓
[Decision: Which workflow?]
    ↓
WorkflowEngine.execute()
    ↓
WorkflowDefinition → Sequential/Parallel Steps
    ↓
StepExecutor (per step)
    ↓
AgentSpawner → PromptBuilder + ToolRegistry
    ↓
ModelResolver (free/standard/premium)
    ↓
vscode.lm.sendChatRequest()
    ↓
[LLM Response]
    ↓
Validation → [success] → next step
          → [failure] → Retry/Escalation
    ↓
Final Summary → Chat Stream
    ↓
Project Model Updated (for next workflow)
```

---

**Next:** Go to Extension Manifest for package.json configuration.