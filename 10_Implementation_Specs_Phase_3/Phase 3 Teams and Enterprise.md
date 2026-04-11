# Phase 3 - Teams and Enterprise

## Roadie for Organizations

**Status:** VISION - Explicitly deferred. Do not design in detail until Phase 2.5 proves solo value.
**Prerequisite:** Phase 2.5 shipped, 100+ solo developer installs, validated product-market fit
**Target:** Extend Roadie from a solo developer tool to a team-wide AI configuration platform
**Estimated Build Time:** 40-60 hours (largest phase)
**Milestones:** M34-M42

---

## Why Phase 3 Is Deferred

Every design decision in Phases 1-2.5 optimizes for a single developer on a single project. Teams introduce:

- **Multi-user state:** whose edits take priority when two developers modify the same generated file?
- **Cross-repo consistency:** how do you keep 50 repos' instructions aligned without being rigid?
- **Permission models:** who can change the team's AI configuration? Who can override it?
- **Scale:** the SQLite single-file database doesn't scale to concurrent team access
- **Distribution:** VS Code extension marketplace is individual-install; teams need org-wide deployment

Designing for these problems before proving the solo experience is excellent is premature optimization. Phase 3 only makes sense when:

1. Solo developers love Roadie (retention > 60% at 30 days)
2. Edit tracking data shows the learning loop actually improves output quality
3. Multiple developers in the same org independently install Roadie and ask "can our whole team use this?"

Signal #3 is the trigger. If nobody asks for team features, don't build them.

---

## What Phase 3 Would Add

### 1. GitHub App Distribution

**Current (Phase 1-2):** Individual developer installs VS Code extension from marketplace.

**Phase 3:** Organization admin installs a GitHub App. The app:
- Automatically generates `.github/` configuration for every repo in the org
- Runs as a GitHub Action (no VS Code needed - CI/CD integration)
- Provides an org-wide dashboard of AI configuration status
- Enforces org-level configuration policies (security rules, model preferences)

**Architecture:** The GitHub App is a thin wrapper around the existing MCP server (Phase 2) running in a GitHub Actions runner. The core engine is identical - only the distribution and trigger mechanism change.

```
Phase 1-2: Developer -> VS Code Extension -> Core Engine -> .github/ files
Phase 3:   GitHub App -> GitHub Action Runner -> Core Engine -> .github/ files (per repo)
```

---

### 2. Org-Level Configuration Hierarchy

**Problem:** In a team, some AI configuration should be consistent across all repos (security review rules, model tier policies, commit conventions), while other configuration should be repo-specific (tech stack, project structure, local patterns).

**Solution:** Three-tier configuration hierarchy:

```
Org level:     .github-org/roadie-config.yml    <-- shared across all repos
    | inherits
Team level:    .github/roadie-team.yml           <-- per-team overrides
    | inherits
Repo level:    .github/.roadie/config             <-- per-repo (existing Phase 1.5 config)
```

**Merge semantics:**
- Org config provides defaults
- Team config overrides org defaults
- Repo config overrides team defaults
- Individual developer `settings.json` overrides repo config (existing Phase 1 behavior)
- Org admin can mark certain settings as `locked: true` - repo-level cannot override

**Example org config:**
```yaml
# .github-org/roadie-config.yml
roadie:
  security:
    reviewTier: premium          # always use premium model for security reviews
    locked: true                 # repos cannot downgrade this
  modelPreference: balanced
  workflowHistory: true          # enabled for all repos
  conventions:
    commitFormat: conventional
    branchNaming: "feature/{ticket}-{description}"
```

---

### 3. Cross-Repo Consistency Engine

**Problem:** An org with 50 repos wants consistent AI instructions. But each repo has different tech stacks, so instructions can't be identical - they need to be consistently structured while content-specific.

**Solution:** A Consistency Engine that:
- Scans all repos in the org
- Identifies common patterns (most repos use TypeScript, most use Vitest, etc.)
- Generates org-level instruction templates that individual repos inherit
- Flags repos that deviate from org patterns (not to enforce - to inform)
- Produces an org-wide AI configuration report

**New MCP tools (Phase 3):**

| Tool | Description |
|------|-------------|
| `roadie/org_scan` | Scan all repos in an org, build cross-repo model |
| `roadie/org_consistency_report` | Compare configuration across repos, flag deviations |
| `roadie/org_generate_templates` | Generate org-level instruction templates from common patterns |
| `roadie/org_sync` | Push org-level config to all repos |

---

### 4. Team Codebase Dictionary

**Current (Phase 1.5):** Each repo has its own Codebase Dictionary in SQLite.

**Phase 3:** Cross-repo entity graph. The dictionary understands relationships BETWEEN repos:
- Service A's API endpoint is consumed by Service B's client
- Shared library C is imported by 12 repos
- Schema changes in DB repo D affect 5 downstream services

**Architecture decision (deferred):** This is where a graph database (Neo4j or similar) might justify its complexity. A cross-repo entity graph with 50 repos x 5000 entities x 15000 relationships approaches 750K nodes - still feasible in SQLite but queries get slow. Evaluate when the data volume is real.

**Storage options (evaluate at build time):**
- Option A: Centralized SQLite file in org config repo (simple, limited scale)
- Option B: PostgreSQL hosted service (scales, but adds cloud dependency - violates Phase 1-2 philosophy)
- Option C: Neo4j (natural for graph queries, heavyweight)
- Option D: Each repo keeps local SQLite, org scan aggregates on demand (no centralized store)

**Recommendation:** Start with Option D (aggregation on demand). No new database. The org scan tool reads each repo's existing SQLite dictionary and builds a temporary in-memory graph for the consistency report. Only move to a centralized store if query latency becomes a real problem.

---

### 5. Multi-User Edit Conflict Resolution

**Problem:** Developer A and Developer B both modify `.github/copilot-instructions.md`. Roadie's append-below merge works for one developer + Roadie. It doesn't work for multiple developers + Roadie.

**Solution:** Version vector per section:

```typescript
interface SectionVersion {
  sectionId: string;
  lastModifiedBy: 'roadie' | string;  // developer username or 'roadie'
  lastModifiedAt: string;
  contentHash: string;
  version: number;  // monotonically increasing per section
}
```

**Conflict resolution policy:**
1. If only Roadie modified since last sync -> apply Roadie's update
2. If only one developer modified -> keep developer's version, append Roadie's below (existing behavior)
3. If multiple developers modified -> keep the most recent developer's version, append others' changes as comments, flag for manual resolution
4. Org admin can set policy: "developer A's edits always take priority" (ownership model)

**This requires git integration:** The extension needs to read git blame to determine who modified each section. Phase 1-2 doesn't use git for attribution - only for detecting changes. Phase 3 adds git blame as a data source.

---

### 6. Shared Skills and Agent Library

**Problem:** Team members create useful custom agents and skills. There's no way to share them.

**Solution:** An org-level skill/agent registry:
- Shared skills live in a central `.github-org/roadie-skills/` repo
- Teams can publish skills from any repo to the central registry
- Roadie's generator pulls from the registry when generating agent definitions
- Version control on shared skills (semver tagging)

---

### 7. Team Dashboard

**Current (Phase 1.5):** Sidebar view showing project model for a single repo.

**Phase 3:** Web-based dashboard (GitHub Pages or similar) showing:
- All repos in the org and their AI configuration status
- Configuration coverage (which repos have instructions, agents, skills?)
- Workflow execution stats across the org (aggregate, anonymous)
- Quality scores per repo (from Phase 2.5 adaptive learning)
- Deviation alerts ("repo X hasn't regenerated instructions in 30 days")

**Technology:** Static HTML + GitHub API. No backend server. Dashboard reads from each repo's `.github/.roadie/` directory via GitHub API and aggregates client-side.

---

## Architecture Impact Assessment

### What Doesn't Change
- Core engine (workflow engine, agent spawner, intent classifier, project model)
- File generation pipeline (section manager, templates, generators)
- MCP server interface (existing 10 tools continue to work)
- VS Code extension (individual developer experience unchanged)
- SQLite for per-repo storage

### What Changes
- Distribution: GitHub App added alongside VS Code extension
- Configuration: three-tier hierarchy replaces flat config
- Git integration: blame data added as input to merge decisions
- MCP tools: 4 new org-level tools added
- Dashboard: new web-based UI (separate from VS Code extension)

### What's Risky
- **Multi-user merge** is the hardest problem. The append-below strategy that works beautifully for solo developers becomes confusing with 5 developers' changes appended below each other.
- **Cross-repo scanning** at org scale (50+ repos) could be slow. Need rate limiting for GitHub API calls.
- **Org-level config locked settings** create a tension between team consistency and developer autonomy. Get the default wrong and developers will fight the tool.

---

## Milestones (Rough - Will Be Refined After Phase 2.5)

### M34: GitHub App Scaffold
**What:** GitHub App registration, webhook handling, Action runner integration.
**Estimated time:** 5-6 hours

### M35: Org Configuration Hierarchy
**What:** Three-tier config with inheritance and locked settings.
**Estimated time:** 6-8 hours

### M36: Org Scan Tool
**What:** `roadie/org_scan` MCP tool that reads all repos and builds cross-repo model.
**Estimated time:** 5-6 hours

### M37: Cross-Repo Consistency Engine
**What:** `roadie/org_consistency_report` and `roadie/org_generate_templates` tools.
**Estimated time:** 6-8 hours

### M38: Multi-User Edit Resolution
**What:** Git blame integration, version vectors per section, conflict resolution policy.
**Estimated time:** 8-10 hours (hardest module in Phase 3)

### M39: Shared Skill/Agent Registry
**What:** Central repo for shared skills, publishing mechanism, version control.
**Estimated time:** 4-5 hours

### M40: Team Dashboard
**What:** Static web dashboard reading from GitHub API.
**Estimated time:** 5-6 hours

### M41: Org Sync Tool
**What:** `roadie/org_sync` MCP tool that pushes org config to all repos.
**Estimated time:** 3-4 hours

### M42: Enterprise Packaging and Documentation
**What:** GitHub Marketplace listing for GitHub App, enterprise documentation, security review.
**Estimated time:** 3-4 hours

---

## Key Technical Decisions (Deferred - Decide at Build Time)

| # | Decision | Options | Evaluate When |
|---|----------|---------|---------------|
| D17 | Cross-repo storage | SQLite aggregation vs PostgreSQL vs Neo4j | When org scan data volume is known |
| D18 | Dashboard technology | GitHub Pages + client-side vs hosted backend | When dashboard requirements are firm |
| D19 | Config lock enforcement | Advisory (warn) vs mandatory (block) | When org admin feedback is available |
| D20 | Multi-user merge | Append-below-per-user vs last-writer-wins vs CRDT | When multi-user edit patterns are observed |
| D21 | GitHub App scope | Read-only vs read-write | Based on security review requirements |

---

## What Must Be True Before Starting Phase 3

1. **Solo experience is proven.** Retention > 60% at 30 days for individual developers.
2. **Learning loop works.** Phase 2.5 adaptive learning demonstrably improves generation quality over time.
3. **Multiple developers in the same org ask for team features.** Organic demand, not speculative.
4. **The core engine is stable.** No major architectural changes needed - Phase 3 extends, doesn't refactor.
5. **Security model is audited.** Enterprise adoption requires a security review of the shell spawn policy, path traversal prevention, and data handling.

If any of these conditions is not met, Phase 3 is premature. Keep improving the solo experience instead.

---

## The Business Case (Brief)

Phase 1-2 is a free VS Code extension that generates value for individual developers. There's no revenue model - it's an adoption play.

Phase 3 introduces the first potential revenue path:
- GitHub App could be freemium (free for public repos, paid for private repos)
- Org-level features (dashboard, consistency engine, shared registry) are enterprise value
- Usage-based pricing on org scan and sync operations

But: **do not design the revenue model until Phase 2.5 proves the product.** Premature monetization design distorts product decisions.

---

## For Agents Reading This

**Do not build Phase 3.** This document is a vision statement, not a build spec. It exists to:
1. Record the team/enterprise direction so the idea isn't lost
2. Ensure Phase 1-2.5 architecture doesn't accidentally block Phase 3
3. Provide context for why certain Phase 1 decisions were made (e.g., provider abstraction enables GitHub Action runner, SQLite enables per-repo isolation)

When Phase 3 is ready to build, each milestone will get a detailed spec at the same level of precision as Phase 1's Module Build Order. That spec doesn't exist yet because it can't - the requirements depend on real usage data from Phases 1-2.5.
