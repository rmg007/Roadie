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
├─ Mon: Create M16 spec (Project Model Persistence)
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

**Status:** ✅ 5/5 COMPLETE  

**Timeline:** ~10-13 hours  

**Blocker For:** Everything else

#### Week 1 - Monday

- [x]  **A1:** Create M16: Project Model Persistence spec (4-5h)
    - [x]  SQLite schema (tech_stack, directory_structure, commands tables)
    - [x]  Load/save/validate operations
    - [x]  Incremental updates from File Watcher
    - [x]  Query APIs for generators
    - [x]  Migration from Phase 1
    - [x]  Build prompt for AI agent
    - **Status:** ✅ COMPLETE

#### Week 1 - Tuesday-Wednesday

- [x]  **FW-1:** Fix File Watcher API (4-5h)
    - [x]  Rewrite change classification (remove addDir/unlinkDir)
    - [x]  Update event handling (onDidCreate/Change/Delete)
    - [x]  Add directory inference logic
    - [x]  Update error handling
    - [x]  Rewrite tests
    - **Status:** ✅ COMPLETE

#### Week 1 - Wednesday-Thursday

- [x]  **SM-1:** Fix Section Manager merge algorithm (2-3h)
    - [x]  Remove "User Priority" option
    - [x]  Implement "Append Below" strategy
    - [x]  Add separator format (timestamp)
    - [x]  Update test cases
    - [x]  Add concurrent edit handling
    - **Status:** ✅ COMPLETE

#### Week 1 - Thursday

- [x]  **C1:** Verify all chokidar references removed
    - [x]  Architecture Overview: ✓ Done
    - [x]  File Watcher spec: Done with FW-1
- [x]  **BC-1:** Verify Phase 1.5 activation boundary documented
    - [x]  Architecture Overview: ✓ Done
    - [x]  All specs reference it: ✓ Verified

### PHASE 2: MUST FIX (Before build starts)

**Status:** ✅ 8/8 COMPLETE  

**Timeline:** ~8-10 hours  

**Blocker For:** AI agent build starts

#### Week 1 - Friday

- [x]  **C4:** Verify Intent Classifier interface (1h)
    - [x]  Check Phase 1 Module Specifications page
    - [x]  Verify uses parseClassification() not classifyWithLLM()
    - [x]  Update if needed
    - **Status:** ✅ COMPLETE
- [x]  **FW-2:** Git checkout optimization (2h)
    - [x]  Smart event batching (100-1000 events)
    - [x]  Watch .git/HEAD for git operations
    - [x]  Debounce git checkout by 2 seconds
    - [x]  Add test case
    - **Status:** ✅ COMPLETE
- [x]  **FW-4:** Add startup reconciliation (1h)
    - [x]  Compare mod times at activation
    - [x]  Process changes as if watcher events
    - [x]  Update File Watcher spec
    - **Status:** ✅ COMPLETE

#### Week 2 - Monday

- [x]  **SM-3:** Replace backup file strategy (1h)
    - [x]  Remove .roadie-new.md backup file creation
    - [x]  Append to original file instead
    - [x]  Add warning about removed markers
    - [x]  Update test cases
    - **Status:** ✅ COMPLETE
- [x]  **SM-4:** Add critical test cases (2h)
    - [x]  Concurrent edit test
    - [x]  Large file test (>1MB)
    - [x]  Encoding test
    - [x]  Section ID collision test
    - [x]  Race condition test
    - **Status:** ✅ COMPLETE
- [x]  **BC-2:** ProjectModel interface extension (1h)
    - [x]  Review Phase 1 ProjectModel interface
    - [x]  Create PersistentProjectModel extends interface
    - [x]  Update all specs to use extension pattern
    - [x]  Add migration guide
    - **Status:** ✅ COMPLETE

#### Week 2 - Tuesday

- [x]  **C2:** Verify config defaults fixed ✓
    - [x]  editTracking: false (not true) ✓
    - [x]  workflowHistory: false (not true) ✓
    - **Status:** ✅ COMPLETE
- [x]  **A3:** Verify database consolidation ✓
    - [x]  Single .github/.roadie/project-model.db ✓
    - [x]  Contains model + learning tables ✓
    - **Status:** ✅ COMPLETE

---

### PHASE 3: SHOULD FIX (During/after build)

**Status:** ✅ 11/11 COMPLETE  

**Timeline:** ~12-25 hours  

**Blocker For:** Phase 1.5 Polish

#### Quick Wins (2 hours)

- [x]  **SM-2:** Document hash normalization (0.5h)
- [x]  **A2:** Add module numbering table (0.5h)
- [x]  **PERF-1:** Add timing assertions (1h)

**When:** Week 2 - anytime

#### Optimization (3 hours)

- [x]  **PERF-3:** Enforce async I/O in linter (2h)
- [x]  **FW-3:** Optimize polling fallback (2h)
- [x]  **PERF-2:** Verify latency math (1h)

**When:** Week 2-3, or during implementation

#### Specs (13 hours)

- [x]  **Generator specs (M25-M32):** 7-8 hours
    - [x]  M25: Copilot Instructions (1h)
    - [x]  M26: Path Instructions (1h)
    - [x]  M27: Agent Definition (1h)
    - [x]  M28: Skill (1h)
    - [x]  M29: Hooks (1h)
    - [x]  M30: Workflows (1h)
    - [x]  M31: Templates (1h)
    - [x]  M32: AGENTS.md (0.5h)
- [x]  **Edit Tracker spec (M21):** 3-4 hours
- [x]  **Learning Database spec (M23):** 3-4 hours
- [x]  **Phase 1 Integration guide:** 2-3 hours

**When:** Week 2, in parallel with AI agent build

---

## Weekly Schedule

### Week 1: Blocking Issues

**Goal:** Fix all BLOCKING issues to unblock AI agent

| Day | Task | Owner | Hours | Status |
| --- | --- | --- | --- | --- |
| **Mon** | Create M16 spec | [TBD] | 4-5 | ✅ DONE |
| **Tue** | FW-1 rewrite (part 1) | [TBD] | 2 | ✅ DONE |
| **Wed** | FW-1 rewrite (part 2) | [TBD] | 2-3 | ✅ DONE |
| **Wed** | SM-1 fix (part 1) | [TBD] | 1 | ✅ DONE |
| **Thu** | SM-1 fix (part 2) | [TBD] | 1-2 | ✅ DONE |
| **Thu** | C1, BC-1 verification | [TBD] | 0.5 | ✅ DONE |
| **Fri** | Buffer/reviews | [TBD] | 1 | ✅ DONE |

**Total:** ~10-13 hours  

**Buffer:** 1 hour for reviews/fixes

### Week 2: MUST FIX Issues + AI Agent Build Starts

**Goal:** Complete all MUST FIX items, then hand off to AI agent

| Day | Task | Owner | Hours | Status |
| --- | --- | --- | --- | --- |
| **Mon** | C4 verify (1h) + FW-2 (2h) + FW-4 (1h) | [TBD] | 4 | ✅ DONE |
| **Mon** | SM-3 (1h) + SM-4 (2h) | [TBD] | 3 | ✅ DONE |
| **Tue** | BC-2 (1h) + buffer | [TBD] | 1-2 | ✅ DONE |
| **Tue** | **AI AGENT STARTS:** M16 implementation | [AI Agent] | 12-16 | ✅ DONE |
| **Tue-Fri** | Quick wins (SM-2, A2, PERF-1) | [TBD] | 2 | ✅ DONE |
| **Tue-Fri** | Parallel: Create M19-27 specs | [TBD] | 7-8 | ✅ DONE |

**Total:** ~30-35 hours (includes AI agent build start)

### Week 3-4: AI Agent Build + Polish

**Goal:** Complete Phase 1.5 implementation

| Task | Owner | Hours | Status |
| --- | --- | --- | --- |
| M16-M29 implementation | AI Agent | 40-50 | ✅ DONE |
| M21 (Edit Tracker) spec | [TBD] | 3-4 | ✅ DONE |
| M23 (Learning DB) spec | [TBD] | 3-4 | ✅ DONE |
| Phase 1 integration guide | [TBD] | 2-3 | ✅ DONE |
| Optimization (PERF-3, FW-3, PERF-2) | [TBD] | 3-4 | ✅ DONE |
| Integration testing | [AI Agent + TBD] | 5-8 | ✅ DONE |
| Bug fixes & polish | [TBD] | 3-5 | ✅ DONE |

**Total:** ~60-78 hours

---

## Dependencies & Blockers

```
M16 (Project Model Persistence)
  ↓ blocks everything
  ├─ M15 (File Watcher) - needs to update model
  ├─ M19 (File Generator Manager) - needs query APIs
  ├─ M25-M32 (Generators) - need model queries
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

1. Fix M16 spec (1 day)
2. Fix FW-1, SM-1 (2-3 days)
3. Fix MUST FIX batch (3-4 days)
4. Hand off to AI agent (can start building)

**Total to unblock AI:** ~1 week

---

## Success Criteria

### For Specification Work

- ✅ All 5 BLOCKING issues fixed
- ✅ All 8 MUST FIX issues fixed
- ✅ M16 spec is clear and detailed
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
- All BLOCKING issues resolved (5/5)
- All MUST FIX issues resolved (8/8)
- All SHOULD FIX issues resolved (11/11)
- All remaining specs created (M19-27, M21, M23)
- Phase 1 fully implemented
- Phase 1.5 fully implemented

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

- A) One AI agent builds sequentially (M16, M15, M19, ...) — slower
- B) Multiple AI agents, different modules in parallel — faster, needs coordination

**Recommendation:** Option B (parallelize if possible, saves 1-2 weeks)

---

## Next Actions

All Phase 1 and Phase 1.5 work is complete. Next step is Phase 2 implementation.

---

**Overall Status:** ✅ COMPLETE — Phase 1 & Phase 1.5 fully implemented (2026-04-12)