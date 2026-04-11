# 📖 Key Decisions & Rationale Log

## Why We Made Each Major Decision — Context for Future Agents and Contributors

This page captures the reasoning behind key architectural and product decisions. Any agent or contributor working on Roadie should read this to understand not just WHAT was decided, but WHY — so they don't re-open settled questions.

---

## D1: Architecture — Option 3 (Extension + Generated Files)

**Decision:** Build Roadie as a VS Code Extension that generates .github/ configuration files.

**Options Considered:**

| Option | What | Rejected Because |
| --- | --- | --- |
| 1. Pure .[agent.md](http://agent.md) | Static Markdown agent files | Can't watch files, can't persist state, can't trigger on events. Only runs when developer prompts it. Hits ceiling quickly. |
| 2. VS Code Extension only | Extension with Chat Participant, no file generation | Intelligence dies when extension is uninstalled. Not portable to CLI/Cloud/other IDEs. |
| 3. Extension + Generated Files | Extension is the brain; generated files are exported knowledge | ✅ **Chosen.** Zero-intervention (file watching), workflow control (TypeScript), learning (state persistence), AND portability (files work everywhere). |
| 4. MCP Server only | Tools exposed via MCP protocol | Can't control conversation flow. Agent decides when to call tools. Less control over UX. |
| 5. All of the above | Extension + MCP + agents + hooks | Too complex for v1. Option 3 + MCP as Phase 2 achieves the same result incrementally. |

**Key insight:** The extension handles everything requiring code (file watching, state, telemetry, learning). The generated files handle everything requiring AI (reasoning, code generation). Each does what it's best at.

---

## D2: Language — TypeScript Everywhere

**Decision:** TypeScript for the extension shell, core engine, and MCP server.

**Rust was seriously considered** for the core engine (project analyzer, workflow orchestrator, file generator). Rust would give 10-50x faster file scanning, a single binary with zero dependencies, cross-platform correctness at compile time, and memory safety for long-running processes.

**TypeScript was chosen because all code is written by AI agents.** Every major coding AI (Claude, GPT, Gemini, Codex) has been trained on vastly more TypeScript than Rust. The quality gap in generated code is significant:

- AI agents write production-quality TypeScript with proper error handling and good patterns
- AI agents write Rust that fights the borrow checker, uses unnecessary `clone()`, misuses lifetimes
- AI agents maintain TypeScript codebases more reliably over months of evolution
- TypeScript's type system is well understood by AI; Rust's ownership model is not

**The performance argument weakens for this specific product.** Roadie scans config files, generates Markdown, and orchestrates LLM calls. It's not a database engine or game renderer. Node.js is fast enough, and correct code matters more than fast code when AI writes it.

**If Rust expertise existed on the team,** the recommendation would have been Rust core + TypeScript shell. But since AI agents are the only developers, TypeScript everywhere maximizes their productivity.

---

## D3: Development Approach — AI Agents Write All Code

**Decision:** No human code. All code written by Claude Code, Codex, or similar AI coding agents.

**Implications for architecture:**

- All modules < 300 lines (fits AI context window)
- Explicit TypeScript interfaces at every module boundary
- Comprehensive tests as guardrails (AI agents need test failures to catch regressions)
- JSDoc headers as context (AI reads these before modifying code)
- `AGENTS.md` at repo root describes architecture for AI agents
- Zod validation at all boundaries (catches AI-generated type errors at runtime)

**Implications for the spec documents:**

- Specs must be unambiguous — AI agents implement literally, don't design
- Each module has a "Build Prompt" ready to paste into Claude Code
- Each module has explicit test cases the agent must implement
- Interface contracts are the source of truth, not prose descriptions

---

## D4: Target User — Solo Developers Only

**Decision:** v1.0 targets solo developers on Copilot Pro/Pro+. No teams, no enterprises.

**Rationale:**

- Solo developers make adoption decisions instantly (no procurement, no security review)
- Solo developers are the best beta testers (direct feedback, no organizational layers)
- Team features (shared configuration, cross-repo consistency, org-level learning) add massive complexity
- Better to build a tool that one developer loves than a tool that satisfies a committee
- Team/enterprise is explicitly Phase 3+ (deferred, not cancelled)

---

## D5: Phasing — Phase 1 → 1.5 → 2 (No Phase 3)

**Decision:** Three phases, each independently valuable. Phase 3 (teams/enterprise) is not planned.

| Phase | What | Why This Order |
| --- | --- | --- |
| 1 (Active Mode) | Chat Participant + Workflow Engine + Lazy Project Model | Proves the core value: "developer chats, magic happens." Must work before anything else matters. |
| 1.5 (Passive Mode) | File watching + Persistent model + Silent file generation + Learning | Makes Roadie truly invisible. Developer never thinks about it. Depends on Phase 1 being solid. |
| 2 (MCP Server) | Cross-tool compatibility | Exports Phase 1+1.5 capabilities to Claude Code, Gemini CLI, etc. Only worth building after the capabilities exist. |

**The critical sequencing rule:** Spec Phase 1 → BUILD Phase 1 → Spec Phase 1.5 → BUILD Phase 1.5 → Spec Phase 2 → BUILD Phase 2. Don't spec all phases before building. Building reveals design flaws that should feed into the next spec.

---

## D6: File Watcher — VS Code FileSystemWatcher (Not chokidar)

**Decision:** Use VS Code's built-in FileSystemWatcher instead of chokidar.

**Rationale:**

- Integrates with VS Code workspace trust automatically
- Works in Remote Development (SSH, WSL, containers) — chokidar doesn't
- Zero additional dependencies
- Managed by VS Code's lifecycle (disposed automatically)

**Tradeoff:** No `addDir`/`unlinkDir` events. Directory changes must be inferred from file creation/deletion paths. This adds complexity to the File Watcher spec but is worth it for the Remote Dev compatibility.

---

## D7: Merge Strategy — Append Below (Not Overwrite, Not Keep-User)

**Decision:** When a developer edits inside a Roadie-owned section and Roadie regenerates, the new content is appended below the human-edited content with a `<!-- roadie:merged:timestamp -->` separator.

**Options rejected:**

- **Overwrite:** Silently destroys human edits. Unacceptable.
- **Keep user version:** Silently discards Roadie's new content. Developer never knows what Roadie wanted to update.
- **Three-way merge:** Risks corrupting content when auto-merging code/config.

**Append-below is the safest option:** Both versions are visible. Developer reconciles manually. Zero data loss on either side.

---

## D8: Configuration Defaults — Everything False

**Decision:** Every boolean configuration option defaults to `false`.

**Design principle:** "Roadie does nothing the developer hasn't implicitly or explicitly consented to." This is a trust-building strategy. The tool works fully with all defaults. Turning things on is an explicit choice. This matters for:

- Privacy (telemetry off by default)
- Predictability (no surprise behaviors)
- Enterprise adoption (security teams trust tools that don't do things without permission)

---

## D9: Single SQLite Database

**Decision:** All data (project model + learning data + section hashes) lives in one SQLite file: `.github/.roadie/project-model.db`.

**Rejected:** Separate `learning.db` file. Multiple databases add complexity (cross-database transactions, two connections, two files to backup/restore) without benefit. A single database with multiple table groups is simpler and more robust.

---

## D10: Intent Classifier — Double-Duty Pattern

**Decision:** LLM classification piggybacks on the first response (one call serves both classification AND response), not a separate LLM call.

**Interface:** `parseClassification(responseText)` + `getClassificationPromptPrefix()` instead of `classifyWithLLM(prompt, model)`.

**Rationale:** A separate classification call doubles the latency and cost of every prompt. The double-duty pattern adds a JSON prefix to the system prompt, and the classifier parses it from the response. One LLM call, two purposes.

---

## D11: Notion as Shared Editing Surface

**Decision:** Use Notion as the collaborative editing surface for all specification documents, with GitHub repo as the eventual canonical source for code.

**Rationale:** Multiple Claude chat agents need to read and write the same documents. Notion MCP enables this. The tradeoff (no version control, no diffs, no branching) is acceptable for specification documents that are reviewed by humans before becoming implementation specs.

**Long-term plan:** Move to GitHub repo when implementation starts. Notion remains the working draft surface; repo is the canonical source.