# AGENT ENTRYPOINT

**Welcome, Coding Agent (Cursor/Windsurf/Antigravity).**
This is the root mapping file for the Roadie repository. Do not guess where to start. Follow this exact sequence to build context and execute tasks.

---

## 🛑 PRE-FLIGHT: The Notion-to-Local Export Pipeline
**For the Human Developer:** Any changes made in Notion **must** be exported as Markdown and overwritten in this local directory `C:\dev\Roadie\Roadie_Project_Documentations_Only` before assigning tasks to the agent.
1. Export the modified Notion page(s) as Markdown.
2. Unzip and place them into the correct directory folder here.
3. **CRITICAL:** Ensure that there are absolutely **no generic Notion links** remaining. Any cross-references must use absolute or relative file URI paths (e.g., `[Spec name](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/...)`).
4. Once the local files are fresh, you may begin your agent prompt.

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
When you are ready to build, always defer to the Module Build Order. 
[Module Build Order & Verification](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/03_Implementation_Specs_Phase_1/Module%20Build%20Order%20&%20Verification.md)

### Agent Execution Rules:
*   **Step-by-Step Isolation:** Implement exactly one module/step at a time. Do not anticipate or write code for future steps unless explicitly shared in the immediate context.
*   **Thin Vertical Slice First:** Step 9 of the Build Order establishes the first "Bug Fix Workflow" end-to-end. Your priority is to ensure the mock infrastructure (Step 3) and data persistence layers seamlessly orchestrate into this vertical slice before expanding to horizontal workflows.
*   **Context Clearing:** Clear your context between major architectural *milestones* (e.g., after the initial Types & Mocks milestone, before moving to the persistent databases). However, **do not clear context between tightly coupled modules** (e.g., M14 Project Model Persistence and M15 File Watcher), as they share interfaces that you must keep in working memory.

---

## 🛠️ 3. Future Actions & Deferred Architecture
### Error Glossary / Self-Healing Map
*Status: Deferred.*
- We intentionally defer creating the `Error Glossary` until **after** Build Order Step 4 (Model Resolver) is fully implemented. We will log actual build/runtime errors and form a true mapping of causes and fixes, preventing fictional hallucinated solutions.

### CI/CD and Security Tooling
*Status: Final Step.*
- Implementation of GitHub Actions, dependency-cruiser, and ESLint security hooks are deferred until the feature work wraps. See [Security and Migration Specification](file:///c:/dev/Roadie/Roadie_Project_Documentations_Only/07_Patterns_and_Standards/Security%20and%20Migration%20Specification.md) for strict rules.
