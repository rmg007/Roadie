# VS Code Copilot — Dynamic Model Selection Guide for AI Coding Agents
**Updated:** April 2026 · **Plan context:** Copilot Pro (300 PRs/mo) or Pro+ (1,500 PRs/mo)  
**Purpose:** Routing logic an orchestrator agent uses to pick the right model per-task, maximizing output while minimizing premium request (PR) burn.

> **Multipliers change.** Before relying on any specific number, verify against
> [GitHub's official multiplier table](https://docs.github.com/en/copilot/reference/ai-models/supported-models).
> The values below are accurate as of April 2026 but are explicitly subject to change.

---

## 1. The Economics — Why This Matters

On Copilot Pro you get **300 premium requests/month**. That's roughly 15 per working day. One careless session with a 3× model burns 30–60 PRs. A single interaction with a high-multiplier reasoning model can consume 10–50 PRs. Meanwhile, the three **included models cost literally nothing** on paid plans — unlimited use, no PR deduction.

The goal is simple: **do 80% of work on 0× models, stretch the remaining 20% across the cheapest premium models that can handle the task, and reserve expensive models for genuinely hard problems.**

---

## 2. Model Tiers by Cost

### Tier 0 — Free / Included (0× multiplier, unlimited on paid plans)

These models consume **zero premium requests** on any paid Copilot plan. They are your workhorse defaults. An agent should route here unless there's a specific reason not to.

| Model | Strengths | Best For |
| :--- | :--- | :--- |
| **GPT-5 mini** | Fast, capable, good reasoning for its size, strong at chain-of-thought | Default for almost everything: chat, explanations, moderate debugging, code generation, iteration |
| **GPT-4.1** | Reliable general-purpose, solid code understanding | Fallback baseline, everyday Q&A, code explanations, simple refactors |
| **GPT-4o** | Multimodal (vision support), fast | UI screenshot analysis, image-based debugging, any task involving visual input |
| **Grok Code Fast 1** | Extremely fast generation, optimized for code | Boilerplate, scaffolding, CRUD, first drafts, file stubs — speed over depth |
| **Raptor mini** *(Preview)* | Fast, lightweight | Quick micro-tasks, trivial code generation |

**Agent rule:** *Always start here. Only escalate when the task demonstrably exceeds these models' capabilities.*

### Tier 1 — Budget Premium (0.33× multiplier)

Each interaction costs ~⅓ of a premium request. On 300 PRs/month, that's up to **~900 effective interactions** — roughly 45/day.

| Model | Strengths | Best For |
| :--- | :--- | :--- |
| **Gemini 3 Flash** *(Preview)* | Fast, cheap, good for bulk work | Mass test generation, repetitive transforms, batch scripting, data processing code |

**Agent rule:** *When the free models feel too shallow but the task is high-volume or repetitive, route here. Best for "write 20 unit tests" style work where throughput matters more than nuance.*

### Tier 2 — Standard Premium (1× multiplier)

Each interaction costs exactly 1 premium request. On 300 PRs/month, that's 15/day.

| Model | Strengths | Best For |
| :--- | :--- | :--- |
| **Claude Sonnet 4.6** | Excellent code quality, strong at UI/UX, clean React/CSS/Tailwind, understands design intent | Frontend polish, component architecture, "make this feel premium," aesthetics-sensitive code |
| **Claude Sonnet 4.5** | Strong all-round coding, reliable | General coding tasks that need more sophistication than free models |
| **Claude Haiku 4.5** | Fast, cheap for Claude family | Quick Claude-quality answers without Sonnet cost — good for subagent research tasks |
| **GPT-5.2** | Strong general-purpose reasoning, balanced speed/quality | Standard feature implementation, moderate debugging, mixed-language codebases |
| **GPT-5.4** | Latest GPT, improved reasoning | When you want the newest OpenAI capabilities at standard cost |
| **GPT-5.1** | Solid predecessor, well-tested | Reliable alternative if 5.4 feels unstable on your task type |
| **Gemini 2.5 Pro** | 1M+ token context window | Massive codebase analysis, cross-repo search, "find every usage of X across the entire project" |

**Agent rule:** *This is where the real cost/quality tradeoff lives. Use Claude Sonnet 4.6 for frontend, Gemini 2.5 Pro for huge-context tasks, and GPT-5.2/5.4 for everything else that the free tier can't handle.*

### Tier 3 — Premium (3× multiplier)

Each interaction costs 3 premium requests. On 300 PRs/month, that's only **100 total interactions** (~5/day).

| Model | Strengths | Best For |
| :--- | :--- | :--- |
| **Claude Opus 4.5** | Deep reasoning, holds complex mental models, traces subtle cross-file interactions | Multi-file architecture refactors, obscure bugs spanning layers, system design |
| **Claude Opus 4.6** | Latest Opus, strongest reasoning available | The hardest problems: "understand the entire flow," "find why this edge case breaks," security audits of complex systems |

**Agent rule:** *Only when the task explicitly requires deep reasoning across many files and simpler models have failed or would clearly be insufficient. Never for boilerplate, explanations, or single-file changes.*

### Tier 4 — Expensive / Specialized (10×+ multiplier)

These burn premium requests extremely fast. A handful of interactions can consume your monthly allowance.

| Model | Multiplier | Best For |
| :--- | :--- | :--- |
| **Claude Opus 4.6 (fast mode)** *(Preview)* | Very high (exact multiplier subject to change) | Production-blocking emergencies where developer time > PR cost |
| **GPT-5.1-Codex-Max** | High | Maximum code generation quality for mission-critical output |
| **o3 / o4-mini** *(reasoning models)* | Varies (high) | Mathematical reasoning, formal verification, algorithmic complexity analysis |

**Agent rule:** *Almost never. Reserve for true emergencies or one-off critical tasks. If the user explicitly says "time is money" or "this is blocking production," escalate here. Otherwise, use Opus 4.6 at 3× first.*

---

## 3. The Agent Decision Algorithm

An agent should run this checklist top-to-bottom, taking the **first match**:

```
STEP 0: Can I answer from context/memory without a model call?
  → Yes: Skip the call entirely. Zero cost.

STEP 1: Is this a trivial/micro task?
  (Boilerplate, scaffolding, comment generation, simple rename, docs update,
   "explain this error," single-line fix, CRUD endpoint)
  → GPT-5 mini (0×) or Grok Code Fast 1 (0×)

STEP 2: Is this a bulk/batch task?
  (Generate 10+ tests, transform many files identically, write repetitive
   data processing code, mass rename/refactor)
  → Gemini 3 Flash (0.33×)

STEP 3: Does this involve analyzing a very large context?
  (Whole-repo search, cross-package refactor, "find all usages of X,"
   understanding a 50+ file codebase, reading extremely long files)
  → Gemini 2.5 Pro (1×) — its 1M token window is unmatched

STEP 4: Is this UI/UX, frontend, or design-sensitive?
  (React components, CSS/Tailwind styling, "make this look modern,"
   component architecture, design system work)
  → Claude Sonnet 4.6 (1×)

STEP 5: Is this a screenshot or image-based task?
  (Read text from screenshot, analyze UI mockup, debug visual layout)
  → GPT-4o (0×) — only production model with full vision support

STEP 6: Is this a standard coding task that free models can't handle well?
  (Feature implementation needing solid reasoning, moderate debugging,
   API integration, algorithm implementation)
  → GPT-5.2 or GPT-5.4 (1×)
  → Or try GPT-5 mini (0×) first — it's surprisingly capable. Escalate
     only if quality is insufficient.

STEP 7: Is this genuinely hard?
  (Multi-file architecture refactor, subtle cross-layer bug, deep system
   design, security audit of complex auth flow, "understand the entire flow")
  → Claude Opus 4.6 (3×)
  → Only after confirming simpler models can't handle it.

STEP 8: Is this a production emergency?
  (Blocking deployment, costing real money per minute of downtime)
  → Claude Opus 4.6 fast mode or GPT-5.1-Codex-Max
  → Accept the PR burn. Developer time > premium requests.
```

---

## 4. Cost Optimization Strategies

### 4.1 Default to Free — Always

Set GPT-5 mini as the default model in your agent config. It handles 70–80% of typical developer tasks at zero cost. Only escalate when you hit a quality wall.

```yaml
# In your orchestrator agent:
model: 'GPT-5 mini'  # Free. Always start here.
```

### 4.2 Use Auto Mode for the 10% Discount

When you let Copilot's Auto model selector choose, **all models get a 10% multiplier discount** on paid plans. Claude Sonnet 4.6 drops from 1× to 0.9×, Opus from 3× to 2.7×. Over a month, this adds up.

**Caveat:** Auto excludes models with multipliers >1× and won't select Opus. Use Auto for routine work, manual selection only when you need a specific premium model.

### 4.3 Use Subagents with Cheap Models for Research

When the orchestrator needs information before acting, delegate to a subagent running a free model:

```yaml
# Research subagent — uses a free model for exploration
---
name: Researcher
user-invocable: false
model: 'GPT-5 mini'        # 0× — unlimited research
tools: ['search/codebase', 'read_file', 'web/fetch']
---
```

The subagent explores the codebase at zero cost. Only the orchestrator (potentially on a premium model) receives the condensed findings.

### 4.4 Match Model to Subagent Role

```yaml
# Orchestrator: needs strong reasoning to coordinate
model: 'GPT-5.2'                    # 1×

# Planner subagent: needs depth for architecture decisions
model: 'Claude Sonnet 4.6'          # 1×

# Implementer subagent: needs good coding, not genius-level reasoning
model: 'GPT-5 mini'                 # 0× — most code changes don't need premium

# Reviewer subagent: needs careful analysis
model: 'Claude Sonnet 4.5'          # 1×

# Bulk test writer subagent: high volume, doesn't need premium quality
model: 'Gemini 3 Flash'             # 0.33×

# Security reviewer (occasional, critical): needs deep reasoning
model: 'Claude Opus 4.6'            # 3× — justified for security
```

### 4.5 The "Try Free First" Pattern

For any task at step 6 or above in the algorithm, try the free model first. If the output quality is insufficient, escalate:

```
Agent internal logic:
1. Attempt with GPT-5 mini (0×)
2. Evaluate output quality (does it compile? does it address the requirements?)
3. If insufficient → retry with GPT-5.2 (1×)
4. If still insufficient → retry with Claude Opus 4.6 (3×)
```

This wastes a little latency but saves PRs on tasks that turn out to be simpler than they looked.

### 4.6 Budget Awareness by Phase

Structure your month around your PR budget:

| Phase | Days | Strategy |
| :--- | :--- | :--- |
| **Week 1–2** | 1–10 | Normal usage. Mix free + 1× models. Save Opus for real needs. |
| **Week 3** | 11–15 | Check PR balance. If >50% remaining, continue normally. If <50%, tighten to mostly free models. |
| **Week 4** | 16–20 | Conservation mode if low. Free models + Gemini Flash only. Save remaining PRs for emergencies. |
| **Exhausted** | Any | Fallback to included models (GPT-4.1, GPT-5 mini). Still fully functional for most work. |

### 4.7 Avoid These PR Traps

- **Don't use Opus for explanations.** GPT-5 mini explains code just fine at 0×.
- **Don't use premium models for inline suggestions.** Completions on included models are unlimited.
- **Don't run long agentic sessions on Opus.** Each autonomous tool call by the agent doesn't cost PRs, but each user prompt does. Batch your instructions into fewer, more detailed prompts.
- **Don't forget agent mode counts.** Every prompt you send in agent mode costs PRs at the selected model's multiplier, even if the agent makes 20 tool calls autonomously (those tool calls are free).

---

## 5. Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════╗
║  ROUTING CHEAT SHEET — paste into agent instructions            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DEFAULT (always start here):                                    ║
║    GPT-5 mini ............ 0×   general purpose, free            ║
║    Grok Code Fast 1 ..... 0×   boilerplate, scaffolding, fast    ║
║    GPT-4o ............... 0×   anything with images/screenshots  ║
║                                                                  ║
║  CHEAP BULK WORK:                                                ║
║    Gemini 3 Flash ....... 0.33× mass tests, batch transforms     ║
║                                                                  ║
║  STANDARD PREMIUM (use sparingly):                               ║
║    Claude Sonnet 4.6 .... 1×   UI/UX, frontend, design-sensitive ║
║    GPT-5.2 / 5.4 ........ 1×   solid coding, standard features  ║
║    Gemini 2.5 Pro ....... 1×   huge context (1M tokens)          ║
║                                                                  ║
║  DEEP REASONING (use rarely):                                    ║
║    Claude Opus 4.6 ...... 3×   hard bugs, architecture, security ║
║                                                                  ║
║  EMERGENCY ONLY:                                                 ║
║    Opus 4.6 fast / o3 ... 10×+ production blockers only          ║
║                                                                  ║
║  AUTO MODE = 10% discount on all multipliers (paid plans)        ║
║  INCLUDED MODELS (0×) = unlimited, no PR cost on paid plans      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 6. Model Selection in Agent Frontmatter

### Per-Agent Model Assignment

```yaml
# Orchestrator — coordinates, doesn't code. Needs good judgment.
---
name: Orchestrator
model: 'GPT-5.2'                       # 1× — good reasoning at standard cost
tools: ['agent', 'search', 'read']
agents: ['Planner', 'Coder', 'Reviewer']
---

# Coder — writes code. Free model handles most implementation.
---
name: Coder
model: 'GPT-5 mini'                    # 0× — free for all coding
tools: ['edit_file', 'create_file', 'terminal', 'search/codebase', 'read_file']
---

# Frontend Specialist — CSS/React/Tailwind work.
---
name: FrontendDev
model: 'Claude Sonnet 4.6'             # 1× — worth it for design quality
tools: ['edit_file', 'create_file', 'read_file', 'search/codebase']
---

# Hard Problems Only — deep debugging, architecture.
---
name: SeniorArchitect
model: 'Claude Opus 4.6'               # 3× — only invoked for genuinely hard tasks
user-invocable: false                   # Subagent only — no accidental usage
tools: ['search/codebase', 'read_file', 'web/fetch']
---
```

### Model Fallback Lists

Use arrays to try cheaper models first:

```yaml
model: ['GPT-5 mini', 'GPT-5.2', 'Claude Sonnet 4.6']
# Tries free model first, falls back to premium only if unavailable
```

---

## 7. What Your Draft Got Wrong (Corrections)

For reference, here's what changed from the original draft:

| Draft Claim | Actual |
| :--- | :--- |
| GPT-5.2 / GPT-5.4 at 1× is "the default balanced choice" | **GPT-5 mini at 0× should be the default.** GPT-5.2/5.4 are 1× — use only when free models fall short. |
| GPT-5 mini at "0× / unlimited" for "quality secondary to velocity" | Correct on cost, wrong on framing. **GPT-5 mini is genuinely capable**, not a quality compromise. It should be the primary workhorse, not a fallback. |
| Grok Code Fast at 0.5× | **Grok Code Fast 1 is 0×** on paid plans — it's free, not half-price. Even better than the draft suggested. |
| "Raptor / Goldeneye (2×) for legacy migration" | There is no model called "Goldeneye." **Raptor mini exists as a 0× preview model.** It's not specialized for legacy migration. |
| Claude Opus 4.6 Fast at 30× | The exact multiplier for fast mode is **subject to change** and varies. Community reports range from 10× to 30×+. Don't hardcode a number — just route here only for emergencies. |
| No mention of Auto mode discount | **Auto mode gives a 10% discount** on all multipliers. Significant savings over manual selection for routine work. |
| No mention of included models being unlimited | **GPT-5 mini, GPT-4.1, and GPT-4o are truly unlimited** on paid plans. This is the single most important cost fact. |
| No mention of GPT-4o for vision | **GPT-4o is the only model with production vision support.** Critical for screenshot-based debugging. |
| Missing Gemini 2.5 Pro's 1M context advantage | **Gemini 2.5 Pro's context window is its killer feature** — use it for whole-repo analysis, not general coding. |
