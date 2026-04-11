# 🔄 Workflow Definitions — Master

## Complete State Machines for All 7 Phase 1 Workflows

---

## Quick Reference

| Workflow | Trigger Intent | Steps | Model Tiers | Key Feature | Page |
| --- | --- | --- | --- | --- | --- |
| Bug Fix | `bug_fix` | 8 | Tier 0→1→2 | Escalation on test failure | [Detail](#bug-fix) |
| Feature Dev | `feature` | 7 | Tier 0→1 | Human approval at step 2 | [Detail](#feature) |
| Refactoring | `refactor` | 5+loop | Tier 0→1 | Incremental with inner loop | [Detail](#refactor) |
| Code Review | `review` | 5∥ | Mix | 5 parallel passes | [Detail](#review) |
| Documentation | `document` | 4 | Tier 0 | Reads code as source of truth | [Detail](#document) |
| Dependency Mgmt | `dependency` | 5 | Tier 0→1 | One-at-a-time upgrades | [Detail](#dependency) |
| Onboarding | `onboard` | 4 | Tier 0 | Architecture overview | [Detail](#onboard) |

---

## BUG FIX WORKFLOW {#bug-fix}

### State Machine

```
INIT
  ↓
Step 1: Locate Error Source [Tier 0]
  ↓ success → Step 2
  ↓ fail → FAIL (no retry)

Step 2: Diagnose Root Cause [Tier 1]
  ↓ success → Step 3
  ↓ fail → FAIL (report diagnosis failed)

Step 3: Generate & Apply Fix [Tier 0→1→2]
  ↓ success → Step 4
  ↓ fail → RETRY (attempts 1-3, escalate per sequence)
  ↓ fail after 3 → Step 4 (skip, report)

Step 4: Verify Fix (Run Tests) [N/A - Shell]
  ↓ pass → Step 5
  ↓ fail → ESCALATE to Step 3 attempt N+1, Tier 1
       (include test output in prompt context)
  ↓ timeout → Same as fail

Step 5: Scan for Sibling Bugs [Tier 0]
  ↓ found → Step 6
  ↓ none → Step 7

Step 6: Fix Siblings [Tier 0→1→2]
  ↓ success → Step 7
  ↓ fail → Continue anyway (Step 7)

Step 7: Add Regression Guard [Tier 0]
  ↓ success → Step 8
  ↓ fail → Continue anyway (Step 8)

Step 8: Generate Summary [Tier 0]
  ↓ → COMPLETE
```

### Step Details

#### Step 1: Locate Error Source

**Agent Role:** `diagnostician`  

**Model Tier:** Tier 0  

**Tools:** file_search, grep, git_log, read_file  

**Timeout:** 30s  

**Input Context:**

- User's error description
- Project model (tech stack, structure)

**Prompt Template:**

```
You are a diagnostic agent. Your job is to LOCATE the source of this error.
Do not diagnose the root cause yet—just find where it occurs.

Error Report:
{error_description}

Project Context:
- Tech Stack: {tech_stack}
- Directory Structure: {directory_tree}
- Recent Changes (git log): {recent_commits}

Tools Available:
- read_file(path): Read file contents
- grep(pattern, path): Search for pattern in files
- git_log(limit=10): Show recent commits

Return ONLY:
1. File path (exact): {path}
2. Line number(s): {line_numbers}
3. Code snippet (5 lines context)
4. Confidence (0.0-1.0)

If you cannot locate the error, report what you tried and why it failed.
```

**Success Criteria:**

- File path found
- Line number(s) identified
- Code snippet shown
- Confidence ≥ 0.7

**Failure Handling:**

- If confidence < 0.7, report "error location uncertain" + best guess
- Continue to Step 2 anyway (diagnostician will refine)

---

#### Step 2: Diagnose Root Cause

**Agent Role:** `diagnostician`  

**Model Tier:** Tier 1 (more nuanced analysis)  

**Tools:** read_file, grep, code_analysis  

**Timeout:** 45s  

**Input Context:**

- Step 1 output (file, line)
- Surrounding code (20 lines context)
- Project model patterns (error handling style, etc.)
- Stack trace (if provided)

**Prompt Template:**

```
You are a diagnostic agent. Your job is to DIAGNOSE the root cause.

Error Location (from Step 1):
File: {file_path}
Lines: {line_numbers}

Code:
{code_snippet}

Error Message:
{error_message}

Stack Trace:
{stack_trace}

Project Patterns (Detected):
- Error Handling: {error_handling_pattern}
- Async Pattern: {async_pattern}
- Null Safety: {null_check_style}

Analyze: What is the ROOT CAUSE of this error?

Return ONLY:
1. Root Cause (2-3 sentences)
2. Contributing Factors (list)
3. Severity (critical/high/medium/low)
4. Fix Difficulty (easy/medium/hard)
5. Similar Patterns in Codebase (if any)

Do not suggest fixes—diagnose only.
```

**Success Criteria:**

- Clear root cause statement
- No speculation
- Factors identified

**Failure Handling:**

- If diagnosis is uncertain, state assumptions made
- Step 3 (fixer) will use diagnosis as context

---

#### Step 3: Generate & Apply Fix

**Agent Role:** `fixer`  

**Model Tier:** Tier 0 (escalate on test failure)  

**Tools:** read_file, edit_file, write_file, lint, format  

**Timeout:** 60s  

**Escalation:** If Step 4 (tests) fails, retry Step 3 with:

- Attempt 2: Same tier, refined prompt + test output
- Attempt 3: Tier 1, test output + diagnostic logging request
- Attempt 4: Tier 1, alternative approach (suggest different fix strategy)
- Attempt 5: Tier 2, comprehensive analysis
- Attempt 6: Report failure

**Prompt Template:**

```
You are a fixer. Your job is to GENERATE AND APPLY a fix.

Diagnosis (from Step 2):
{diagnosis}

Code to Fix:
{code_with_context}

Project Patterns:
- Error Handling: {error_handling}
- Code Style: {code_style}
- Testing: {test_pattern}

Requirements:
1. Fix the root cause
2. Follow project code style/patterns
3. No public API changes
4. Add null checks where appropriate
5. Keep change minimal

Approach:
1. Explain the fix (1-2 sentences)
2. Show BEFORE code
3. Show AFTER code (with comments)
4. Indicate which file(s) to edit

After Step 4 (test verification):
- If tests pass: Done
- If tests fail: {test_output will be provided for retry}
```

**Success Criteria:**

- Code compiles/lints cleanly
- Follows project patterns
- Minimal change
- Comment explains fix

**Failure Handling:** (See escalation sequence above)

---

#### Step 4: Verify Fix

**Agent Role:** None (shell command)  

**Tools:** shell (test runner)  

**Timeout:** `roadie.testTimeout` (default 300s)  

**Command:** Detected from project model (e.g., `npm run test`, `pytest`, `go test`)

**Logic:**

```
function verifyFix(testCommand):
  result = shell.run(testCommand, timeout: roadie.testTimeout)
  
  if result.exitCode === 0:
    return {status: 'success', output: result.stdout}
  else:
    // Tests failed—trigger escalation
    return {status: 'failed', stderr: result.stderr, stdout: result.stdout}
end
```

**Success Criteria:**

- Test runner exits with code 0
- No timeout

**Failure Handling:**

- **Test failure:** Escalate to Step 3, Attempt 2+ with test output
- **Timeout:** Treat as failure, escalate
- **No tests:** Skip to manual verification note (Phase 1.5 feature)

---

#### Step 5-8: (Brief)

**Step 5:** Grep codebase for similar bug patterns. Return list of files.  

**Step 6:** For each similar file, apply the same fix. Tier 0→1 escalation.  

**Step 7:** Generate a test that would catch this bug. Add to test suite.  

**Step 8:** Summarize what was done. Return final report to chat.  

### Prompt Templates Per Step

[See detailed templates in "Workflow Prompt Templates" page]

---

## FEATURE DEVELOPMENT WORKFLOW {#feature}

### State Machine

```
INIT
  ↓
Step 1: Analyze Requirements [Tier 0]
  ↓ → Step 2

Step 2: Present Plan for Approval [Tier 0→1]
  ↓ user clicks "Approve" → Step 3
  ↓ user clicks "Revise" → Step 1 (loop with feedback)
  ↓ timeout (5 min) → PAUSED (wait indefinitely)

Step 3: Delegate to Layer Agents [Tier 0→1, parallel]
  ├─ Database Agent
  ├─ Backend Agent
  ├─ Frontend Agent
  ↓ all complete → Step 4
  ↓ one fails → retry independently (up to 2x)

Step 4: Integrate Layers [Tier 0]
  ↓ → Step 5

Step 5: Run Tests [N/A]
  ↓ pass → Step 6
  ↓ fail → PAUSED (report to developer)

Step 6: Quality Review [Tier 1]
  ↓ pass → Step 7
  ↓ fail → PAUSED (report findings)

Step 7: Generate Commit Messages [Tier 0]
  ↓ → COMPLETE
```

### Branching: Plan Approval

**Decision Tree (Step 2):**

```
┌─ Generate plan
├─ Show in chat with [Approve] [Revise] buttons
├─ Wait for user response (indefinite timeout)
│  ├─ Approve → continue to Step 3
│  └─ Revise
│      └─ User provides feedback in message
│      └─ Loop to Step 1 with feedback context
└─ If user closes chat → Workflow persists, can resume
```

**Plan Approval Prompt Template:**

```
You are a planner. Your job is to CREATE A FEATURE PLAN.

Feature Request:
{user_request}

Project Context:
- Tech Stack: {tech_stack}
- Architecture: {architecture_summary}
- Existing Patterns: {patterns}

Create a detailed plan:

## Plan

### Database Layer
- Schema changes (if any)
- Migrations
- Expected impact

### Backend Layer
- New endpoints
- Business logic
- Database queries

### Frontend Layer
- New components
- State management
- Styling

### Timeline Estimate
- Database: X min
- Backend: Y min
- Frontend: Z min
- Total: ~X+Y+Z min

### Risks / Assumptions
- [list]

Present this plan to the developer for approval or revision.
```

---

## REFACTORING WORKFLOW {#refactor}

### State Machine (with Inner Loop)

```
Step 1: Analyze Structure [Tier 0]
  ↓ → Step 2

Step 2: Write Characterization Tests [Tier 1]
  ↓ tests pass → Step 3
  ↓ fail → FAIL (cannot capture current behavior)

Step 3: Refactor Incrementally [Loop]
  ├─ Generate incremental change (extract function, rename, etc.)
  ├─ Apply change
  ├─ Run tests
  │  ├─ Pass → continue loop (next refactoring)
  │  └─ Fail → REVERT change, try alternative
  └─ No more refactorings → exit loop → Step 4

Step 4: Generate Summary [Tier 0]
  ↓ → COMPLETE
```

### Inner Loop Pseudocode

```
function refactorIncrementally(module, characterizationTests):
  remaining_refactorings = [
    "extract_duplicate_logic",
    "split_large_function",
    "simplify_conditionals",
    "improve_variable_names"
  ]
  
  applied = []
  
  while remaining_refactorings.length > 0:
    refactoring = remaining_refactorings.shift()
    
    // Generate change
    change = generateRefactoring(module, refactoring)
    before_hash = hash(module.code)
    
    // Apply change
    module.code = applyChange(module.code, change)
    
    // Test
    result = runTests(characterizationTests)
    
    if result.pass:
      applied.push({refactoring, change})
      // Validate public API unchanged
      if publicAPIChanged(before_hash, hash(module.code)):
        logWarning("Public API changed—stopping refactoring")
        break
    else:
      // Revert
      module.code = revertChange(module.code, change)
      logFailure(`Refactoring '${refactoring}' failed: ${result.output}`)
      // Try alternative
      if hasAlternative(refactoring):
        remaining_refactorings.unshift(alternativeRefactoring)
end
```

### Key Invariant

**Public API must not change.** If a refactoring would alter exports, the workflow stops and reports this as a blocker.

```
function validatePublicAPI(beforeCode, afterCode):
  exports_before = parseExports(beforeCode)
  exports_after = parseExports(afterCode)
  
  if exports_before !== exports_after:
    throw new Error("Public API changed. Stopping refactoring.")
end
```

---

## CODE REVIEW WORKFLOW {#review}

### State Machine (5 Parallel Passes)

```
INIT
  ↓
Spawn 5 agents in parallel via Promise.allSettled():
  ├─ Pass 1: Security Review [Tier 1]
  ├─ Pass 2: Performance Review [Tier 0]
  ├─ Pass 3: Code Quality Review [Tier 0]
  ├─ Pass 4: Test Coverage Review [Tier 0]
  └─ Pass 5: Standards Review [Tier 0]
  ↓ all complete (or timeout)
  ↓
Step: Consolidate Findings [Tier 0]
  ↓
COMPLETE
```

### Parallel Execution Pattern

```
const reviewPromises = [
  spawnAgent('security_reviewer', 'security pass'),
  spawnAgent('performance_reviewer', 'performance pass'),
  spawnAgent('code_quality_reviewer', 'quality pass'),
  spawnAgent('test_reviewer', 'coverage pass'),
  spawnAgent('standards_reviewer', 'standards pass')
];

const results = await Promise.allSettled(reviewPromises);

// All results available, even if some fail
const findings = results
  .map((r, i) => r.status === 'fulfilled' ? r.value : {agentIndex: i, error: r.reason})
  .filter(f => f) // remove undefined
```

### Consolidation Logic

**Group findings by severity:**

```
const consolidated = {
  critical: [],
  warning: [],
  suggestion: []
};

for each finding in allFindings:
  if finding.severity === 'critical':
    consolidated.critical.push(finding)
  else if finding.severity === 'warning':
    consolidated.warning.push(finding)
  else:
    consolidated.suggestion.push(finding)
end
```

---

## DOCUMENTATION WORKFLOW {#document}

### State Machine

```
INIT
  ↓
Step 1: Identify Documentation Target [Tier 0]
  ↓ → Step 2

Step 2: Read Source Code [Tier 0]
  ↓ code read → Step 3
  ↓ file not found → FAIL (report missing file)

Step 3: Generate Documentation [Tier 0]
  ↓ generated → Step 4
  ↓ fail → RETRY once (same tier)

Step 4: Write Documentation File [N/A - file I/O]
  ↓ written → COMPLETE
  ↓ fail → FAIL (report write error)
```

### Step Details

**Step 1 — Identify Target**
- **Role:** `documentarian`
- **Tier:** Tier 0
- **Tools:** `file_search`, `grep`
- **Input:** User's documentation request
- **Prompt Template:**
  ```
  Identify the documentation target from this request: {user_request}
  
  Return:
  1. File(s) or module to document (exact paths)
  2. Documentation type: README | JSDoc | API spec | inline comments
  3. Scope: single function | entire module | public API
  ```
- **Success:** File path(s) identified

**Step 2 — Read Source Code**
- **Role:** `documentarian`
- **Tier:** Tier 0
- **Tools:** `read_file`
- **Input:** File path(s) from Step 1
- **Action:** Read the source files. No LLM call required — just file I/O.

**Step 3 — Generate Documentation**
- **Role:** `documentarian`
- **Tier:** Tier 0
- **Timeout:** 45s
- **Prompt Template:**
  ```
  You are a technical writer. Write documentation for the following code.
  
  Documentation Type: {doc_type}
  Scope: {scope}
  
  Source Code:
  {source_code}
  
  Project Context:
  - Tech Stack: {tech_stack}
  - Conventions: {patterns}
  
  Requirements:
  1. Accurate — reflects actual code behavior (not assumed behavior)
  2. Complete — covers all public methods/properties
  3. Follows project documentation style
  4. Include code examples if type is README or API spec
  5. JSDoc: use {param} and {returns} tags
  ```
- **Success:** Documentation text produced

**Step 4 — Write File**
- Write documentation to file. If JSDoc: edit source file in-place. If README: write `.md` file.
- Use VS Code `workspace.applyEdit()` for file writes.
- **Success:** File written, no write error

---

## DEPENDENCY MANAGEMENT WORKFLOW {#dependency}

### State Machine

```
INIT
  ↓
Step 1: Audit Current Dependencies [Tier 0]
  ↓ → Step 2

Step 2: Identify Target Updates [Tier 0→1]
  ↓ updates identified → Step 3
  ↓ all up-to-date → COMPLETE (report)

Step 3: Update One Dependency [Tier 0]
  [loop per dependency, one at a time]
  ↓ updated → Step 4

Step 4: Verify (Run Tests) [N/A - shell]
  ↓ pass → loop back to Step 3 (next dependency)
  ↓ fail → PAUSED (report breaking change to developer)

Step 5: Generate Summary Report [Tier 0]
  ↓ → COMPLETE
```

### Step Details

**Step 1 — Audit Current Dependencies**
- **Role:** `project_analyzer`
- **Tier:** Tier 0
- **Tools:** `read_file`
- **Action:** Read `package.json`, scan lock file for pinned versions.
- **Prompt Template:**
  ```
  Audit the current dependency state.
  
  package.json contents:
  {package_json}
  
  Return:
  1. All dependencies with current version
  2. Which are outdated (check against known latest versions if possible)
  3. Any CVE/vulnerability signals (look for flagged packages)
  4. Which have known breaking changes in next major
  ```

**Step 2 — Identify Target Updates**
- **Role:** `project_analyzer`
- **Tier:** Tier 1 (may need to reason about breaking changes)
- **Prompt Template:**
  ```
  Given this dependency audit:
  {audit_results}
  
  User request: {user_request}
  
  Identify which packages to update. Order from lowest-risk to highest-risk.
  For each: note if there are known breaking changes (semver major bump).
  
  Return prioritized update list: [{package, from_version, to_version, risk: low|medium|high}]
  ```

**Step 3 — Update One Dependency (Loop)**
- **Role:** `fixer`
- **Tier:** Tier 0
- **Tools:** `shell` (run package manager update command)
- **Action per iteration:**
  ```
  command = packageManager + ' install ' + packageName + '@' + targetVersion
  shell.run(command, timeout: 60s)
  ```
- **If high-risk package:** Prompt user for confirmation before running (via `stream.button()`)

**Step 4 — Verify After Each Update**
- **Action:** Run project test command
- **On Fail:** PAUSE workflow, report breaking change details to developer
- **On Pass:** Continue loop to next dependency

**Step 5 — Summary**
- Report: packages updated, versions changed, any skipped due to failures.

---

## ONBOARDING WORKFLOW {#onboard}

### State Machine

```
INIT
  ↓
Step 1: Read Project Model [Tier 0]
  ↓ model loaded → Step 2

Step 2: Generate Architecture Overview [Tier 0]
  ↓ → Step 3

Step 3: Generate Getting-Started Guide [Tier 0]
  ↓ → Step 4

Step 4: Stream Summary to Developer [N/A]
  ↓ → COMPLETE
```

### Step Details

**Step 1 — Read Project Model**
- **Action:** Load `ProjectModel` (already available via extension context, no extra I/O needed).
- **No LLM call required.**

**Step 2 — Generate Architecture Overview**
- **Role:** `documentarian`
- **Tier:** Tier 0
- **Timeout:** 45s
- **Prompt Template:**
  ```
  You are onboarding a new developer to this project.
  
  Project Context:
  - Tech Stack: {tech_stack}
  - Directory Structure: {directory_tree}
  - Key Commands: {commands}
  - Detected Patterns: {patterns}
  
  Developer's request: {user_request}
  
  Generate a clear architecture overview covering:
  1. What this project does (1 paragraph)
  2. Tech stack and why each is used
  3. Directory structure map (annotated)
  4. Data flow for the most important path(s)
  5. Key files / entry points to read first
  ```

**Step 3 — Generate Getting-Started Guide**
- **Role:** `documentarian`
- **Tier:** Tier 0
- **Prompt Template:**
  ```
  Using the architecture overview:
  {architecture_overview}
  
  Generate a practical getting-started guide:
  1. Setup steps (install, config, env)
  2. Run command: {dev_command}
  3. Test command: {test_command}
  4. First task recommendation for a new contributor
  5. Key contacts / documentation links (if detectable from project files)
  ```

**Step 4 — Stream to Developer**
- Stream the combined architecture overview + getting-started guide to chat.
- No file write unless user explicitly requests it.

---

## Prompt Template Repository

All step-specific prompt templates are maintained in the local file: `06_Workflows_and_Prompts/Workflow Prompt Templates Repository.md`

Update that file when tuning prompts for better results.

---

**Next:** See "Workflow Prompt Templates" for complete prompt text for all steps.