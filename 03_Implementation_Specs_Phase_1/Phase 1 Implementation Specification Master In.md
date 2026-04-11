# 📋 Phase 1 Implementation Specification — Master Index (SUPERSEDED)

> ## ⚠️ SUPERSEDED — DO NOT IMPLEMENT FROM THIS FILE
>
> This was an early outline of the Phase 1 spec. The canonical build runbook is `03_Implementation_Specs_Phase_1/Module Build Order & Verification.md`, which contains the 13-step ordered Build Prompt plan with all fixes applied (verbatim types, inlined mock classes, inlined fixtures, etc.).
>
> **Agents:** read `00_START_HERE.md` at workspace root, then `Module Build Order & Verification.md`. Do NOT implement from this file.
>
> **Superseded on:** 2026-04-11

**Status:** SUPERSEDED  
**Version:** 1.0 (historical)  
**Target:** AI Coding Agents (Claude Code, Codex)  
**Date Created:** April 2026

---

## Overview

This specification contains **everything an AI coding agent needs** to implement Phase 1 of Roadie. Phase 1 delivers Active Mode — the Chat Participant, intent classification, workflow engine, agent spawning, and lazy project model. It is the core product experience.

When Phase 1 is complete:

- Developers install → see nothing → select Roadie from chat dropdown → type natural language → workflows execute → magic happens
- No file watching, no persistent state, no learning database, no MCP server (all Phase 1.5+)
- One workflow execution = one session; project model rebuilt on each chat (lazy)

---

## What Phase 1 Delivers

✅ **Active Mode:** Chat Participant receives prompts, intent classifier routes them, workflows execute  

✅ **Seven Workflows:** Bug fix, feature development, refactoring, code review, documentation, dependency management, onboarding  

✅ **Intent Classification:** Two-tier (local + LLM fallback) with 8 intent types  

✅ **Workflow Engine:** State machine with sequential/parallel execution, retry/escalation, cancellation  

✅ **Agent Spawner:** Creates ephemeral subagents with role-specific prompts, scoped tools, model selection  

✅ **Lazy Project Model:** Node.js/TypeScript only; rebuilt on first workflow of session; no persistence  

✅ **File Generation:** `.github/copilot-instructions.md` and `AGENTS.md` (no passive regen)  

✅ **Model Escalation:** Three-tier cost hierarchy (free → standard → premium)  

✅ **Basic Configuration:** Settings in `.vscode/settings.json` (testTimeout, modelPreference, telemetry, etc.)  

---

## What Phase 1 Does NOT Include

❌ Passive mode (file watching, automatic regeneration)  

❌ Persistent project model across sessions  

❌ Edit tracking or learning database  

❌ Section ownership markers or merge logic  

❌ Sidebar UI or status dashboard  

❌ Extended file generation (per-language instructions, agent definitions, skills, hooks)  

❌ MCP server  

❌ Multi-ecosystem support (only Node.js/TypeScript in v1.0)  

---

## Document Structure

This spec is organized into **10 major sections**, each with detailed sub-pages:

### 1. **Module Specifications** (14 modules in Phase 1)

Complete spec for each module: identity, interface contract, internal design, implementation notes, test spec.  

→ *Pages:* Module Specifications — Index + 14 child pages (one per module)

### 2. **Module Build Order**

Exact sequence to build modules, accounting for dependencies. "Build prompt" for each step ready to hand to Claude Code.  

→ *Page:* Module Build Order & Verification

### 3. **Workflow Definitions**

State machines, transitions, prompt templates, tool scoping, model tier assignments, escalation logic for all 7 workflows.  

→ *Page:* Workflow Definitions — Master + child pages per workflow

### 4. **Intent Classification Taxonomy**

8 intent types, trigger keywords, confidence thresholds, example prompts, fallback behavior.  

→ *Page:* Intent Classification Taxonomy

### 5. **Model Selection Strategy**

Tier hierarchy, per-step assignments, escalation triggers, cost budgeting, fallback behavior.  

→ *Page:* Model Selection Strategy

### 6. **End-to-End Integration Scenarios**

5 complete scenarios from user prompt → final result. Includes happy path + failure recovery.  

→ *Page:* Integration Scenarios

### 7. **Extension Manifest**

Complete `package.json` contributions: Chat Participant, commands, settings, activation events.  

→ *Page:* Extension Manifest & Configuration

### 8. **Project Structure**

Complete file/folder layout, module assignments, test co-location, where workflows/templates live.  

→ *Page:* Phase 1 Project Structure

### 9. **Shared Types & Interfaces**

All TypeScript interfaces used across modules. Single source of truth for module contracts.  

→ *Page:* Shared TypeScript Interfaces

### 10. **Implementation Patterns & Standards**

Code conventions, error handling patterns, testing patterns, what AI agents should follow/avoid.  

→ *Page:* Implementation Patterns & Standards

---

## How to Use This Spec

**For AI Coding Agents:**

1. Read this index page and the "Implementation Patterns & Standards" page
2. Read the "Shared TypeScript Interfaces" page
3. Go to the Module Build Order page and start with the first module
4. For each module, fetch its dedicated page and implement per the spec
5. Use the "End-to-End Integration Scenarios" page to verify behavior

**For Code Review:**

1. Module Specifications → verify implemented interface matches contract
2. Test Specification section → verify tests exist and cover scenarios
3. Integration Scenarios → run end-to-end tests against the scenarios

**For Debugging:**

- Module Specifications → understand what a module does and its inputs/outputs
- Module Build Order → see dependencies and what must be built first
- End-to-End Scenarios → trace through the full flow

---

## Key Design Principles

1. **No Ambiguity:** Every design decision is made in this spec. AI agents implement, don't design.
2. **Module Independence:** Each module has a clear interface contract. Implementation is internal.
3. **Cost-Awareness:** Tier 0 (free) by default. Escalate to Tier 1 only on failure. Tier 2 is last resort.
4. **Error Propagation:** Errors surface to developers. Workflows don't silently fail.
5. **TypeScript First:** All interfaces in TypeScript. Zod validation for all cross-module data.
6. **Test-Driven:** Tests are part of the spec. Each module has corresponding test cases.

---

## Navigation Quick Links

- [Module Specifications — Index](link)
- [Module Build Order & Verification](link)
- [Workflow Definitions — Master](link)
- [Intent Classification Taxonomy](link)
- [Model Selection Strategy](link)
- [Integration Scenarios](link)
- [Extension Manifest & Configuration](link)
- [Phase 1 Project Structure](link)
- [Shared TypeScript Interfaces](link)
- [Implementation Patterns & Standards](link)

---

## Document Metadata

| Attribute | Value |
| --- | --- |
| **Scope** | Phase 1 (Active Mode Only) |
| **Target Audience** | AI Coding Agents, Senior Engineers |
| **Completeness** | Ready for implementation |
| **Last Updated** | April 2026 |
| **Dependencies** | PDD v1.0, TAD v1.0, Development Roadmap v1.0 |

---

**Next:** Start with [Shared TypeScript Interfaces](link), then [Module Build Order](link), then implement modules in order.

[🔧 Shared TypeScript Interfaces](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%A7%20Shared%20TypeScript%20Interfaces%2033fc821ae63c8110a6afe29efeadd9ef.md)

[📐 Implementation Patterns & Standards](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%93%90%20Implementation%20Patterns%20&%20Standards%2033fc821ae63c81a188f2c4b7b085c997.md)

[📊 Module Build Order & Verification](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%93%8A%20Module%20Build%20Order%20&%20Verification%2033fc821ae63c8138979fe54b6de61503.md)

[🔍 Intent Classification Taxonomy](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%8D%20Intent%20Classification%20Taxonomy%2033fc821ae63c81868e06d3b8d5bdbe85.md)

[🔺 Model Selection Strategy](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%BA%20Model%20Selection%20Strategy%2033fc821ae63c814088eace402c614aad.md)

[🌀 End-to-End Integration Scenarios](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%8C%80%20End-to-End%20Integration%20Scenarios%2033fc821ae63c817db6aee914d9631ce6.md)

[⚙️ Extension Manifest & Configuration](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%E2%9A%99%EF%B8%8F%20Extension%20Manifest%20&%20Configuration%2033fc821ae63c81a591ade4ad10cdd5ba.md)

[📁 Phase 1 Project Structure](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%93%81%20Phase%201%20Project%20Structure%2033fc821ae63c81b5b8e4ec181d4c616b.md)

[📋 Phase 1 Implementation Spec — Complete (MASTER)](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%93%8B%20Phase%201%20Implementation%20Spec%20%E2%80%94%20Complete%20(MASTER)%2033fc821ae63c81408ce8cf2387003603.md)

[🔄 Workflow Definitions — Master](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%84%20Workflow%20Definitions%20%E2%80%94%20Master%2033fc821ae63c817ca733d8eed29a78ca.md)

[📝 Workflow Prompt Templates Repository](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%93%9D%20Workflow%20Prompt%20Templates%20Repository%2033fc821ae63c81468abcc11996e1ab0b.md)

[💼 Module-Specific Implementation Guides](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%92%BC%20Module-Specific%20Implementation%20Guides%2033fc821ae63c81bf9688e83bd617c338.md)

[🔄 Phase 1.5 Implementation Specification — Master Index](%F0%9F%93%8B%20Phase%201%20Implementation%20Specification%20%E2%80%94%20Master%20In/%F0%9F%94%84%20Phase%201%205%20Implementation%20Specification%20%E2%80%94%20Master%20%2033fc821ae63c817c80ecfb0138d892b4.md)