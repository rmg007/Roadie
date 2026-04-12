# Origin Guides

**These eight documents are where Roadie began.**

On April 2, 2026, a developer uploaded `copilot-customization-reference.md` to a Claude chat and asked to improve it. That improvement session expanded into a comprehensive guide series covering every aspect of GitHub Copilot configuration. Nine days later, the question "if we know exactly what good configuration looks like, why make the developer write it?" turned these guides into the product specification for Roadie.

---

## The Guides

| # | Guide | What It Covers |
|---|-------|---------------|
| 1 | **Copilot Customization Reference** | The original document. Complete reference for .github/ configuration files, instruction syntax, agent definitions, skills, hooks, and workflows. The foundation everything else builds on. |
| 2 | **Copilot Instructions Authoring Guide** | How to write effective copilot-instructions.md files. Section structure, tone, specificity, what to include vs omit, per-language instructions, path-specific instructions. |
| 3 | **Copilot Agent Authoring Guide** | How to create custom .agent.md files. Role definition, tool scoping, instruction writing, agent specialization, when to use agents vs skills. |
| 4 | **Copilot Agent Environment Guide** | Decision framework for Local vs Background (CLI) vs Cloud agents. Capabilities, limitations, and routing strategies for each environment. |
| 5 | **Copilot Agents Orchestration Guide** | Multi-agent patterns: handoffs, subagent delegation, orchestrator agents, parallel execution, sequential chains. How to build workflows from agent compositions. |
| 6 | **Copilot Skill Authoring Guide** | How to create reusable .md skill files. Frontmatter format, instruction body, tool declarations, when skills are better than agents. |
| 7 | **Copilot Model Selection Guide** | Model tier hierarchy, when to use which tier, cost awareness, escalation patterns, premium request budgeting for Copilot Pro/Pro+. |
| 8 | **Workspace Optimizer Agent** | A complete, ready-to-use .agent.md file that audits and maintains the .github/ directory. This is both documentation AND a working artifact a developer can copy into any project today. |

---

## How Roadie Uses These Guides

Each guide maps directly to one or more Roadie components:

| Guide | Roadie Component |
|-------|-----------------|
| Customization Reference | Project Analyzer - knows what configuration files exist and their formats |
| Instructions Authoring | Copilot Instructions Generator - produces copilot-instructions.md following the guide's best practices |
| Agent Authoring | Agent Definitions Generator - creates .agent.md files with proper role definitions |
| Environment Guide | Workflow Engine - routes tasks to the right model tier and execution environment |
| Orchestration Guide | Agent Spawner - implements handoff, parallel, and sequential patterns |
| Skill Authoring | Skills Generator - creates skill .md files with correct frontmatter |
| Model Selection | Model Resolver - maps tiers to available models with fallback chains |
| Workspace Optimizer | The conceptual prototype - Roadie IS this agent, automated |

**When building file generators:** Read the corresponding authoring guide to understand what "good" output looks like. The generator's output should match what the guide teaches a human to write.

---

## For Agents Reading This

These guides are **reference material, not build instructions.** They explain the domain knowledge that Roadie automates. You should read the relevant guide before implementing any generator or workflow that produces the same type of content.

Do NOT implement from these guides. Implement from the specs in folders 03-08. But when you need to understand WHY a generator produces a specific section structure, or WHAT makes a good agent definition, these guides have the answer.

---

## The Origin Story

```
April 2, 2026:   "improve/enhance/update this reference"
                      |
                      v
                 8 comprehensive guides produced
                      |
                      v
April 10, 2026:  "if we know what good configuration looks like,
                  why make the developer write it?"
                      |
                      v
                 Product Design Document
                 Technical Architecture Document
                 Development Roadmap
                 42 milestone specifications
                      |
                      v
April 11, 2026:  Zero-Guesswork audit passed
                 Documentation corpus: Mission-Ready
                      |
                      v
                 Ready to build Roadie
```
