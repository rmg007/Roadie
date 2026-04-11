# 📁 Phase 1 Project Structure

# Phase 1 Project Structure

## Complete File & Folder Layout

---

## Directory Tree

```
roadie/
├── .github/
├── .vscode/
├── out/                      # Bundled output (git-ignored)
├── src/
├── test/
├── .eslintrc.json
├── .prettierrc
├─┠ .gitignore
├── CHANGELOG.md              # Version history
├── LICENSE                   # MIT
├── package.json              # Extension manifest + dependencies
├── README.md                 # Marketplace description
├── tsconfig.json             # TypeScript config (ES2022, modules)
├── tsup.config.ts           # Bundler config
├── vitest.config.ts         # Test config
└── AGENTS.md                 # Root-level AI instructions (generated later in Phase 1.5)
```

---

## src/ Directory

```
src/
├── extension.ts              # [M0] Entry: activate/deactivate, DI setup
├── extension.test.ts         # Integration tests
├── types.ts                  # [M0] All shared TypeScript interfaces
├── container.ts              # [M0] Dependency injection container
├── container.test.ts
├──
├── shell/                    # [M0] VS Code API shell layer
├── ├── chat-participant.ts     # Chat Participant handler, routing
├── ├── chat-participant.test.ts
├── ├── status-bar.ts           # Status bar item
├── ├── status-bar.test.ts
├── ├── commands.ts             # [M13] Command palette registrations
├── └── commands.test.ts
├──
├── classifier/               # [M2] Intent classification
├── ├── intent-classifier.ts    # Two-tier classifier
├── ├── intent-classifier.test.ts
├── └── intent-patterns.ts      # Pattern map
├──
├── engine/                   # [M3-M6] Workflow execution
├── ├── workflow-engine.ts      # State machine orchestration
├── ├── workflow-engine.test.ts
├── ├── step-executor.ts        # Step execution + retry/escalation
├── ├── step-executor.test.ts
├── ├── model-resolver.ts       # [M1] Tier-to-model mapping
├── ├── model-resolver.test.ts
├── └── definitions/            # Workflow definitions (one per workflow)
├──     ├── bug-fix.ts            # [M6] Bug fix workflow
├──     ├── bug-fix.test.ts
├──     ├── feature.ts            # [M9] Feature development
├──     ├── feature.test.ts
├──     ├── refactor.ts           # [M10] Refactoring
├──     ├── refactor.test.ts
├──     ├── review.ts             # [M11] Code review
├──     ├── review.test.ts
├──     ├── document.ts           # [M12] Documentation
├──     ├── document.test.ts
├──     ├── dependency.ts         # [M12] Dependency management
├──     ├── dependency.test.ts
├──     ├── onboard.ts            # [M12] Onboarding
├──     └── onboard.test.ts
├──
├── spawner/                  # [M4] Agent spawning
├── ├── agent-spawner.ts        # Create ephemeral agents
├── ├── agent-spawner.test.ts
├── ├── prompt-builder.ts       # Three-layer prompts
├── ├── prompt-builder.test.ts
├── └── tool-registry.ts        # Tool scoping
├──     tool-registry.test.ts
├──
├── analyzer/                # [M5] Project analysis
├── ├── project-analyzer.ts     # Orchestrates analysis
├── ├── project-analyzer.test.ts
├── ├── dependency-scanner.ts   # Reads package.json, lock files
├── ├── dependency-scanner.test.ts
├── ├── directory-scanner.ts    # fast-glob directory scanning
├── └── directory-scanner.test.ts
├──
├── model/                    # [M5] Project model
├── ├── project-model.ts        # In-memory model + debounce
├── ├── project-model.test.ts
├── └── database.ts             # SQLite operations
├──     database.test.ts
├──
├── generator/                # [M7] File generation
├── ├── file-generator.ts       # Orchestrates generation
├── ├── file-generator.test.ts
├── ├── section-manager.ts      # Ownership markers, hashing
├── ├── section-manager.test.ts
├── └── templates/              # Markdown templates
├──     ├── copilot-instructions.ts
├──     └── agent-definitions.ts
└──
```

---

## test/ Directory

```
test/
├── mocks/                    # [M1] Mock infrastructure
├── ├── mock-language-model.ts   # Mock LLM API
├── ├── mock-chat-response-stream.ts
├── └── mock-fixtures.ts         # Canned responses
├──
├── fixtures/                # Real test projects
├── ├── node-js-next-js/         # Next.js + Prisma test project
├── ├── ├── package.json
├── ├── ├── tsconfig.json
├── ├── ├── src/
├── ├── └── __tests__/
├── ├──
├── ├── node-js-express/        # Express.js test project
├── ├── ├── package.json
├── ├── ├── src/
├── ├── └── __tests__/
├── └── python-flask/           # Python test project (optional for v1.0)
├──
├── integration/              # End-to-end scenarios
├── ├── bug-fix.e2e.test.ts
├── ├── feature.e2e.test.ts
├── └── review.e2e.test.ts
└──
├── snapshots/                # Vitest snapshot files
└── __mocks__/               # Jest-style mocks (if used)
```

---

## Module to File Mapping

| Module | File(s) | Milestone | Lines (Est.) |
| --- | --- | --- | --- |
| Extension Shell | `extension.ts`, `container.ts` | M0 | 150 |
| Chat Participant | `shell/chat-participant.ts` | M0 | 200 |
| Status Bar | `shell/status-bar.ts` | M0 | 80 |
| Commands | `shell/commands.ts` | M13 | 120 |
| Intent Classifier | `classifier/intent-classifier.ts` | M2 | 250 |
| Intent Patterns | `classifier/intent-patterns.ts` | M2 | 150 |
| Model Resolver | `engine/model-resolver.ts` | M1 | 120 |
| Workflow Engine | `engine/workflow-engine.ts` | M3 | 280 |
| Step Executor | `engine/step-executor.ts` | M3 | 250 |
| Bug Fix Workflow | `engine/definitions/bug-fix.ts` | M6 | 180 |
| [5 More Workflows] | `engine/definitions/*.ts` | M9-12 | 900 |
| Project Analyzer | `analyzer/project-analyzer.ts` | M5 | 200 |
| Dependency Scanner | `analyzer/dependency-scanner.ts` | M5 | 220 |
| Directory Scanner | `analyzer/directory-scanner.ts` | M5 | 150 |
| Project Model | `model/project-model.ts` | M5 | 280 |
| Database | `model/database.ts` | M5 | 240 |
| Agent Spawner | `spawner/agent-spawner.ts` | M4 | 200 |
| Prompt Builder | `spawner/prompt-builder.ts` | M4 | 220 |
| Tool Registry | `spawner/tool-registry.ts` | M4 | 160 |
| File Generator | `generator/file-generator.ts` | M7 | 240 |
| Section Manager | `generator/section-manager.ts` | M7 | 180 |
| Templates | `generator/templates/*.ts` | M7 | 300 |
| **TOTAL** | **~28 files** | **M0-M13** | **~4500 lines** |

---

## Test Co-Location Pattern

```
src/classifier/intent-classifier.ts
src/classifier/intent-classifier.test.ts

src/engine/workflow-engine.ts
src/engine/workflow-engine.test.ts

[etc for every module]
```

**Benefit:** When modifying a module, its tests are immediately visible (same directory listing).

---

## Configuration Files

### .vscode/launch.json

Debugger configuration for F5 (Extension Development Host):

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Extension",
      "type": "extensionHost",
      "request": "launch",
      "runtimeExecutable": "${execPath}",
      "args": ["--extensionDevelopmentPath=${workspaceFolder}"],
      "outFiles": ["${workspaceFolder}/out/**/*.js"]
    }
  ]
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "declaration": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "outDir": "./out"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.test.ts", "test/**/*"]
}
```

### vitest.config.ts

```tsx
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  },
});
```

---

## .gitignore

```
# Dependencies
node_modules/

# Build output
out/
*.vsix

# Test coverage
coverage/

# Editor
.DS_Store
.vscode/*
!.vscode/launch.json

# Runtime
*.log

# Phase 1.5+
.github/.roadie/project-model.db
.github/.roadie/*.json
```

---

## Total Codebase Size (Phase 1)

- **Source files:** ~28 modules, ~4500 lines of TypeScript
- **Test files:** ~28 test files, ~2000 lines of Vitest
- **Total:** ~6500 lines of code + tests
- **Dependencies:** 2 major (better-sqlite3, zod) + dev tools
- **Build time:** <5 seconds
- **Bundle size:** ~500 KB (uncompressed, .vsix ~200 KB)

---

## Key Design Decisions

1. **Test co-location:** {module}.ts → {module}.test.ts in same directory
2. **No global state:** All modules pass context explicitly
3. **Single types.ts:** Shared types imported everywhere, circular dependencies impossible
4. **Module boundary validation:** All cross-module data validated with Zod
5. **Maximum 300 lines per file:** Forces clean module boundaries
6. **Async/await throughout:** No callbacks, no promise chains

---

**Next:** Create a Module Specifications index page linking to detailed specs for each of the 14 modules.