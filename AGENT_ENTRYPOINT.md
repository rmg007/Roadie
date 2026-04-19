# AGENT ENTRYPOINT

**Welcome, Coding Agent (Claude Code / Cursor / Windsurf).**
This is the root mapping file for the Roadie project.

> ## ✅ Implementation Status — 2026-04-14
>
> **Phase 1 (Active Mode) and Phase 1.5 (Passive Mode) are fully implemented** in `../roadie/`.  
> Phase 2 (MCP Server) is specified but not yet built.  
>
> **Do not re-implement anything in Phase 1 or Phase 1.5.** The code already exists.  
> Focus areas: bug fixes, test improvements, documentation alignment, Phase 2 preparation.

---

## 🛑 PRE-FLIGHT: Documentation is the Source of Truth
Spec documents live locally at `C:\dev\Roadie\Roadie_Project_Documentations_Only`. The Notion workspace is no longer the primary source — all canonical specs are in these local files.

If spec documents are updated in Notion, export them as Markdown and place them in the correct folder before beginning agent work.

---

## 📖 1. Context Acquisition Sequence
Read these core files in this exact order to understand the state of the project, its structure, and its constraints. **Do not begin coding until you have ingested these.**

1. **Architecture Overview:**
   [Phase 1 5 Architecture Overview](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/02_Technical_Architecture/Phase%201%205%20Architecture%20Overview.md)
2. **Implementation Patterns & Standards:**
   [Implementation Patterns & Standards](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/07_Patterns_and_Standards/Implementation%20Patterns%20&%20Standards.md)
3. **Canonical Data Contracts:**
   Read both of these to understand the exact types and validations you must use.
   * [Shared TypeScript Interfaces](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/07_Patterns_and_Standards/Shared%20TypeScript%20Interfaces.md)
   * [Shared Zod Schemas](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/07_Patterns_and_Standards/Shared%20Zod%20Schemas.md)
4. **Security & Migration Rules:**
   [Security and Migration Specification](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/07_Patterns_and_Standards/Security%20and%20Migration%20Specification.md)

---

## 🏗️ 2. Execution Sequence

**Phase 1 and Phase 1.5 are COMPLETE.** For bug fixes or test work, go directly to the relevant source file in `../roadie/src/` and its paired spec in this docs repo.

For **Phase 2** work, the specification is at:
[Phase 2 Implementation Specification](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/05_Implementation_Specs_Phase_2/Phase%202%20Implementation%20Specification%20Master%20In.md)

**Historical reference only** — the original Phase 1 build order (already executed):
[Module Build Order & Verification](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/03_Implementation_Specs_Phase_1/Module%20Build%20Order%20&%20Verification.md)

### Agent Execution Rules:
*   **Do not re-implement Phase 1 or Phase 1.5 modules.** Read the existing code first, then propose targeted changes.
*   **Tests are mandatory.** Any bug fix must include a failing test that demonstrates the bug, then a fix that makes it pass.
*   **One module at a time.** Do not anticipate or write code for unrelated modules.
*   **Context Clearing:** Clear context between major architectural milestones (e.g., after finishing a complete module).

---

## 🛠️ 3. Future Actions & Deferred Architecture
### Error Glossary / Self-Healing Map
*Status: Deferred.*
- We intentionally defer creating the `Error Glossary` until **after** Build Order Step 4 (Model Resolver) is fully implemented. We will log actual build/runtime errors and form a true mapping of causes and fixes, preventing fictional hallucinated solutions.

### CI/CD and Security Tooling
*Status: Final Step.*
- Implementation of GitHub Actions, dependency-cruiser, and ESLint security hooks are deferred until the feature work wraps. See [Security and Migration Specification](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/07_Patterns_and_Standards/Security%20and%20Migration%20Specification.md) for strict rules.
