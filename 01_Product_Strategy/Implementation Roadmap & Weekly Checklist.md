# 🖯 Implementation Roadmap & Weekly Checklist

> ## ⚠️ SUPERSEDED — HISTORICAL SCHEDULING ARTIFACT
>
> This file was the week-by-week work plan for resolving pre-remediation blocking issues. **All BLOCKING and MUST-FIX items listed below are now RESOLVED** (see `02_Technical_Architecture/Issues & Fixes Status Tracking Dashboard.md`).
>
> Task rows that reference "M14" here predate the canonical module ID decision. **The canonical ID for Project Model Persistence is M16** — see `00_START_HERE.md` at workspace root.
>
> **Agents:** do NOT treat this file as a current checklist. The current build runbook is `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`.
>
> **Superseded on:** 2026-04-11

**Date:** April 11, 2026  
**Status:** ✅ BLOCKING ISSUES RESOLVED — corpus is build-ready.  
**Owner:** Autonomous coding agent (Claude Code / Cursor / Windsurf / Antigravity)

---

## Critical Path: Issues That Block Build

```
WEEK 1 (THIS WEEK)
├─ Mon: Create M14 spec (Project Model Persistence)
├─ Tue-Wed: Fix FW-1 (File Watcher API rewrite)
├─ Wed-Thu: Fix SM-1 (Section Manager merge algorithm)
├─ Thu-Fri: Fix MUST FIX batch (6 items)
└─ Result: Ready for AI agent build handoff

WEEK 2
├─ AI Agent: Build Phase 1.5 modules (56-65 hours)
└─ Parallel: Create remaining specs (M19-27, integration guide)

WEEK 3-4
├─ Integration testing
├─ Polish & optimization
└─ Release Phase 1.5
```

---

## Detailed Checklist

### PHASE 1: BLOCKING ISSUES (Must fix first)

**Status:** 🔴 0/5 COMPLETE  

**Timeline:** ~10-13 hours  

**Blocker For:** Everything else

#### Week 1 - Monday

- [ ]  **A1:** Create M14: Project Model Persistence spec (4-5h)
    - [ ]  SQLite schema (tech_stack, directory_structure, commands tables)
    - [ ]  Load/save/validate operations
    - [ ]  Incremental updates from File Watcher
    - [ ]  Query APIs for generators
    - [ ]  Migration from Phase 1
    - [ ]  Build prompt for AI agent
    - **Assigned To:** [TBD]
    - **Expected Completion:** Monday EOD

#### Week 1 - Tuesday-Wednesday

- [ ]  **FW-1:** Fix File Watcher API (4-5h)
    - [ ]  Rewrite change classification (remove addDir/unlinkDir)
    - [ ]  Update event handling (onDidCreate/Change/Delete)
    - [ ]  Add directory inference logic
    - [ ]  Update error handling
    - [ ]  Rewrite tests
    - **Assigned To:** [TBD]
    - **Expected Completion:** Wednesday EOD

#### Week 1 - Wednesday-Thursday

- [ ]  **SM-1:** Fix Section Manager merge algorithm (2-3h)
    - [ ]  Remove "User Priority" option
    - [ ]  Implement "Append Below" strategy
    - [ ]  Add separator format (timestamp)
    - [ ]  Update test cases
    - [ ]  Add concurrent edit handling
    - **Assigned To:** [TBD]
    - **Expected Completion:** Thursday noon

#### Week 1 - Thursday

- [ ]  **C1:** Verify all chokidar references removed
    - [ ]  Architecture Overview: ✓ Already done
    - [ ]  File Watcher spec: Will be done with FW-1
- [ ]  **BC-1:** Verify Phase 1.5 activation boundary documented
    - [ ]  Architecture Overview: ✓ Already done
    - [ ]  All specs reference it: To verify

### PHASE 2: MUST FIX (Before build starts)

**Status:** 🟡 2/8 COMPLETE (C2, A3 fixed)  

**Timeline:** ~8-10 hours  

**Blocker For:** AI agent build starts

#### Week 1 - Friday

- [ ]  **C4:** Verify Intent Classifier interface (1h)
    - [ ]  Check Phase 1 Module Specifications page
    - [ ]  Verify uses parseClassification() not classifyWithLLM()
    - [ ]  Update if needed
    - **Assigned To:** [TBD]
- [ ]  **FW-2:** Git checkout optimization (2h)
    - [ ]  Smart event batching (100-1000 events)
    - [ ]  Watch .git/HEAD for git operations
    - [ ]  Debounce git checkout by 2 seconds
    - [ ]  Add test case
    - **Assigned To:** [TBD]
- [ ]  **FW-4:** Add startup reconciliation (1h)
    - [ ]  Compare mod times at activation
    - [ ]  Process changes as if watcher events
    - [ ]  Update File Watcher spec
    - **Assigned To:** [TBD]

#### Week 2 - Monday

- [ ]  **SM-3:** Replace backup file strategy (1h)
    - [ ]  Remove .[roadie-new.md](http://roadie-new.md) backup file creation
    - [ ]  Append to original file instead
    - [ ]  Add warning about removed markers
    - [ ]  Update test cases
    - **Assigned To:** [TBD]
- [ ]  **SM-4:** Add critical test cases (2h)
    - [ ]  Concurrent edit test
    - [ ]  Large file test (>1MB)
    - [ ]  Encoding test
    - [ ]  Section ID collision test
    - [ ]  Race condition test
    - **Assigned To:** [TBD]
- [ ]  **BC-2:** ProjectModel interface extension (1h)
    - [ ]  Review Phase 1 ProjectModel interface
    - [ ]  Create PersistentProjectModel extends interface
    - [ ]  Update all specs to use extension pattern
    - [ ]  Add migration guide
    - **Assigned To:** [TBD]

#### Week 2 - Tuesday

- [ ]  **C2:** Verify config defaults fixed ✓
    - [ ]  editTracking: false (not true) ✓
    - [ ]  workflowHistory: false (not true) ✓
    - Status: ALREADY DONE
- [ ]  **A3:** Verify database consolidation ✓
    - [ ]  Single .github/.roadie/project-model.db ✓
    - [ ]  Contains model + learning tables ✓
    - Status: ALREADY DONE

---

### PHASE 3: SHOULD FIX (During/after build)

**Status:** 🟢 2/11 COMPLETE (C3, A2 partial)  

**Timeline:** ~12-25 hours  

**Blocker For:** Phase 1.5 Polish

#### Quick Wins (2 hours)

- [ ]  **SM-2:** Document hash normalization (0.5h)
- [ ]  **A2:** Add module numbering table (0.5h)
- [ ]  **PERF-1:** Add timing assertions (1h)

**When:** Week 2 - anytime

#### Optimization (3 hours)

- [ ]  **PERF-3:** Enforce async I/O in linter (2h)
- [ ]  **FW-3:** Optimize polling fallback (2h)
- [ ]  **PERF-2:** Verify latency math (1h)

**When:** Week 2-3, or during implementation

#### Specs (13 hours)

- [ ]  **Generator specs (M20-27):** 7-8 hours
    - [ ]  M20: Copilot Instructions (1h)
    - [ ]  M21: Path Instructions (1h)
    - [ ]  M22: Agent Definition (1h)
    - [ ]  M23: Skill (1h)
    - [ ]  M24: Hooks (1h)
    - [ ]  M25: Workflows (1h)
    - [ ]  M26: Templates (1h)
    - [ ]  M27: [AGENTS.md](http://AGENTS.md) (0.5h)
- [ ]  **Edit Tracker spec (M21):** 3-4 hours
- [ ]  **Learning Database spec (M23):** 3-4 hours
- [ ]  **Phase 1 Integration guide:** 2-3 hours

**When:** Week 2, in parallel with AI agent build

---

## Weekly Schedule

### Week 1: Blocking Issues

**Goal:** Fix all BLOCKING issues to unblock AI agent

| Day | Task | Owner | Hours | Status |
| --- | --- | --- | --- | --- |
| **Mon** | Create M14 spec | [TBD] | 4-5 | 🔴 TODO |
| **Tue** | FW-1 rewrite (part 1) | [TBD] | 2 | 🔴 TODO |
| **Wed** | FW-1 rewrite (part 2) | [TBD] | 2-3 | 🔴 TODO |
| **Wed** | SM-1 fix (part 1) | [TBD] | 1 | 🔴 TODO |
| **Thu** | SM-1 fix (part 2) | [TBD] | 1-2 | 🔴 TODO |
| **Thu** | C1, BC-1 verification | [TBD] | 0.5 | 🔴 TODO |
| **Fri** | Buffer/reviews | [TBD] | 1 | 🔴 TODO |

**Total:** ~10-13 hours  

**Buffer:** 1 hour for reviews/fixes

### Week 2: MUST FIX Issues + AI Agent Build Starts

**Goal:** Complete all MUST FIX items, then hand off to AI agent

| Day | Task | Owner | Hours | Status |
| --- | --- | --- | --- | --- |
| **Mon** | C4 verify (1h) + FW-2 (2h) + FW-4 (1h) | [TBD] | 4 | 🔴 TODO |
| **Mon** | SM-3 (1h) + SM-4 (2h) | [TBD] | 3 | 🔴 TODO |
| **Tue** | BC-2 (1h) + buffer | [TBD] | 1-2 | 🔴 TODO |
| **Tue** | **AI AGENT STARTS:** M14 implementation | [AI Agent] | 12-16 | 🔴 TODO |
| **Tue-Fri** | Quick wins (SM-2, A2, PERF-1) | [TBD] | 2 | 🔴 TODO |
| **Tue-Fri** | Parallel: Create M19-27 specs | [TBD] | 7-8 | 🔴 TODO |

**Total:** ~30-35 hours (includes AI agent build start)

### Week 3-4: AI Agent Build + Polish

**Goal:** Complete Phase 1.5 implementation

| Task | Owner | Hours | Status |
| --- | --- | --- | --- |
| M14-M29 implementation | AI Agent | 40-50 | 🔴 TODO |
| M21 (Edit Tracker) spec | [TBD] | 3-4 | 🔴 TODO |
| M23 (Learning DB) spec | [TBD] | 3-4 | 🔴 TODO |
| Phase 1 integration guide | [TBD] | 2-3 | 🔴 TODO |
| Optimization (PERF-3, FW-3, PERF-2) | [TBD] | 3-4 | 🔴 TODO |
| Integration testing | [AI Agent + TBD] | 5-8 | 🔴 TODO |
| Bug fixes & polish | [TBD] | 3-5 | 🔴 TODO |

**Total:** ~60-78 hours

---

## Dependencies & Blockers

```
M14 (Project Model Persistence)
  ↓ blocks everything
  ├─ M15 (File Watcher) - needs to update model
  ├─ M19 (File Generator Manager) - needs query APIs
  ├─ M20-27 (Generators) - need model queries
  ├─ M21 (Edit Tracker) - needs to record changes
  └─ M23 (Learning DB) - in same database

FW-1 (File Watcher API fix)
  ↓ blocks
  └─ M15 implementation

SM-1 (Section Manager merge fix)
  ↓ blocks
  └─ M22 implementation (data loss risk)
```

**Critical Path:**

1. Fix M14 spec (1 day)
2. Fix FW-1, SM-1 (2-3 days)
3. Fix MUST FIX batch (3-4 days)
4. Hand off to AI agent (can start building)

**Total to unblock AI:** ~1 week

---

## Success Criteria

### For Specification Work

- ✅ All 5 BLOCKING issues fixed
- ✅ All 8 MUST FIX issues fixed
- ✅ M14 spec is clear and detailed
- ✅ Comprehensive Review page is complete
- ✅ Build prompts are ready for AI agent

### For AI Agent Build

- ✅ All tests pass (npm run test)
- ✅ Linter passes (npm run lint)
- ✅ Performance budgets met (< 2s generators, < 1s watcher latency)
- ✅ Phase 1 backward compatibility maintained
- ✅ No data loss scenarios
- ✅ SQLite integrity verified

### For Phase 1.5 Polish

- ✅ All 11 SHOULD FIX items addressed
- ✅ Edge cases handled
- ✅ Integration tests passing
- ✅ Remote Development tested
- ✅ Performance optimized

---

## Current Status

### Completed ✅

- Architecture Overview updated (C1, C2, C3, A3, BC-1)
- Comprehensive Review page created
- Blocking issues documented (3 detail pages)
- Issues & Fixes tracking dashboard created

### In Progress 🟡

- BLOCKING issue fixes (none yet)
- MUST FIX issues (none yet)

### Not Started 🔴

- SHOULD FIX issues (17/22)
- Remaining specs (M19-27, M21, M23)

---

## Resource Planning

### Team Needs

**Specification Work (Week 1-2):**

- 1 person (experienced with VS Code/TypeScript) ≈ 20-25 hours
- Or can be split: spec writer (15h) + reviewer (5-10h)

**AI Agent Build (Week 2-4):**

- 1-2 AI agents (Claude/similar) ≈ 56-65 hours
- Can be parallelized (multiple agents, different modules)

**Testing & Polish (Week 3-4):**

- 1-2 people ≈ 8-10 hours
- Parallel with build

### Total Team Effort

- Specs: 20-25 hours
- Build: 56-65 hours (AI)
- Test/Polish: 8-10 hours
- **Grand Total:** ~84-100 hours over 4 weeks

---

## Decision Points

### Decision 1: Sync vs Async

**Options:**

- A) Fix all specs first (Week 1-2), THEN build (Week 3-4) — 4 weeks total
- B) Fix BLOCKING only (3 days), start build immediately, fix SHOULD FIX in parallel — 3 weeks total

**Recommendation:** Option B (start build early, save 1 week)

### Decision 2: Defer SHOULD FIX?

**Options:**

- A) Fix all SHOULD FIX items before build — 4-5 weeks
- B) Fix only BLOCKING + MUST FIX, add SHOULD FIX during/after build — 3-4 weeks
- C) Defer SHOULD FIX to Phase 1.5.1 — 3 weeks

**Recommendation:** Option B (do SHOULD FIX in parallel with build)

### Decision 3: Parallel AI Builds

**Options:**

- A) One AI agent builds sequentially (M14, M15, M16, ...) — slower
- B) Multiple AI agents, different modules in parallel — faster, needs coordination

**Recommendation:** Option B (parallelize if possible, saves 1-2 weeks)

---

## Next Actions (Immediate)

### Today

1. [ ] Review this checklist with team
2. [ ] Assign owners to Week 1 tasks
3. [ ] Prioritize AI agents for build

### Monday (Start Week 1)

1. [ ] Start M14 spec creation
2. [ ] Begin FW-1 research (VS Code API deep dive)
3. [ ] Begin SM-1 merge algorithm redesign

### By Friday (End Week 1)

1. [ ] All BLOCKING issues fixed
2. [ ] Hand off to AI agent
3. [ ] Start MUST FIX batch

---

**Overall Status:** 🟡 CRITICAL PATH IDENTIFIED, READY TO EXECUTE

**Estimated Time to Phase 1.5 Complete:** 3-4 weeks (with parallel work)