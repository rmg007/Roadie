# 🏷️ Section Manager Specification

## Detects Section Ownership, Computes Hashes, Merges Human Edits with Roadie-Generated Content

---

## Module Identity

**Module ID:** M22  

**File Location:** `src/generator/section-manager.ts`  

**Depends On:** Learning Database (M23)  

**Used By:** File Generator Manager (M19), Edit Tracker (M21), all 9 file generators  

**Complexity:** **CRITICAL** (merge logic is most error-prone in Phase 1.5)  

**Estimated Build Time:** 6-8 hours  

**Implementation Status:** ✅ COMPLETE — Implemented as of 2026-04-12

---

## The Problem

Roadie generates files (.github/[copilot-instructions.md](http://copilot-instructions.md), agents, workflows, etc.). But developers might edit those files. How do we:

1. **Detect** when a developer has edited Roadie-generated content?
2. **Preserve** the developer's edits when Roadie regenerates?
3. **Merge** intelligently if both Roadie and developer changed the same section?
4. **Handle** edge cases (deleted markers, human moved sections, etc.)?

**The solution: Section ownership markers with hash-based change detection.**

---

## Section Ownership Markers

### Marker Format (Different for Each File Type)

#### Markdown Files (.github/copilot-*.md, .github/[AGENTS.md](http://AGENTS.md), etc.)

```markdown
<!-- roadie:start:section-id -->
Roadie-generated content goes here.
This section is automatically updated.
<!-- roadie:end:section-id -->
```

**Marker syntax:**

- Start: `<!-- roadie:start:{section_id} -->`
- End: `<!-- roadie:end:{section_id} -->`
- All content between markers = Roadie-owned
- `section_id` = identifier for this section (e.g., "python-patterns", "dependencies")

#### YAML Files (.github/agents/*.yaml, .github/workflows/*.yml)

```yaml
# roadie:start:section-id
name: bug_fix_agent
description: Handles bug fix workflows
steps:
  - analyze
  - fix
# roadie:end:section-id
```

**Marker syntax:**

- Start: `# roadie:start:{section_id}`
- End: `# roadie:end:{section_id}`
- Comments, not part of YAML structure

#### JSON Files (if any: .github/config.json, etc.)

```json
{
  "roadie_start_tools": true,
  "tools": [
    {
      "name": "read_file"
    }
  ],
  "roadie_end_tools": true
}
```

**Marker syntax:**

- Start: `"roadie_start_{section_id}": true`
- End: `"roadie_end_{section_id}": true`
- Roadie-owned = array/object between markers

#### Shell Scripts (.github/hooks/*.sh)

```bash
#!/bin/bash

# roadie:start:post-commit
echo "Roadie tracking commit..."
# roadie:end:post-commit

# Human-added code below
echo "My custom hook logic"
```

**Marker syntax:**

- Start: `# roadie:start:{section_id}`
- End: `# roadie:end:{section_id}`
- Comments

---

## Hash Tracking

### Why Hashes?

When Roadie regenerates a file, we need to know if the developer has modified the Roadie-owned sections. We can't just compare content (formatting changes, timestamps, etc. will differ). Instead, we hash the content.

### Hash Computation Algorithm

```
function computeSectionHash(content: string): string {
  // 1. Normalize whitespace
  const normalized = content
    .split('\n')
    .map(line => line.trim())
    .filter(line => line.length > 0)
    .join('\n');
  
  // 2. Hash with SHA256
  const hash = crypto.createHash('sha256').update(normalized).digest('hex');
  
  // 3. Return first 8 chars (sufficient for collision detection)
  return hash.substring(0, 8);
end
```

**Why normalize?**

- Ignore formatting changes (developer reformats code)
- Ignore comment-only changes (add a // comment)
- Focus on actual content changes

**Why 8 chars?**

- Short enough to include in comments
- 2^32 possible values = low collision risk

### Hash Storage

Hash is stored in a comment at the END of each section:

```markdown
<!-- roadie:start:dependencies -->
<!-- hash:a1b2c3d4 -->
Roadie-generated content...
<!-- roadie:end:dependencies -->
```

When Roadie regenerates:

1. Extract old hash from comment
2. Compute hash of new content
3. Compare
4. If same = developer didn't edit
5. If different = developer edited

---

## Merge Algorithm

The core logic of Section Manager:

```jsx
function mergeFileContent(
  existingFile: string,
  newContent: string,
  sectionDefinitions: SectionDefinition[]
): string {
  // 1. Parse existing file (find sections)
  const existingSections = parseFile(existingFile);
  const newSections = parseFile(newContent);
  
  const result = [];
  let currentPos = 0;
  
  // 2. For each section in newContent
  for (const newSection of newSections) {
    // Add content BEFORE this section (from existing file)
    const beforeContent = existingFile.substring(currentPos, findSectionStart(existingFile, newSection.id));
    result.push(beforeContent);
    
    // 3. Check if this section was edited by user
    const existingSection = existingSections.find(s => s.id === newSection.id);
    
    if (!existingSection) {
      // Section is new, just add it
      result.push(newSection.toText());
    } else if (hashMatches(existingSection, existingSection.hash)) {
      // User didn't edit this section, use new content
      result.push(newSection.toText());
    } else {
      // User edited this section — APPEND BELOW (never overwrite)
      const timestamp = new Date().toISOString();
      const merged = [
        existingSection.content,
        `<!-- roadie:merged:${timestamp} -->`,
        newSection.content
      ].join('\n');
      result.push(formatSection(newSection.id, merged));
      logMerge(newSection.id, 'append-below');
    }
    
    currentPos = findSectionEnd(existingFile, newSection.id);
  }
  
  // 4. Append any content AFTER last section
  result.push(existingFile.substring(currentPos));
  
  return result.join('\n');
end
```

### Merge Conflict Handling

When a developer edits a Roadie-owned section AND Roadie regenerates it:

**Canonical Strategy: Append Below (from PDD/TAD)**

```jsx
// When human has edited inside a Roadie section:
// 1. Keep the human-edited content as-is
// 2. Append the new Roadie content BELOW, separated by a merge marker
// 3. Developer sees BOTH versions and reconciles manually

function mergeWithAppendBelow(existingContent, newContent, sectionId) {
  const timestamp = new Date().toISOString();
  return [
    existingContent,
    `<!-- roadie:merged:${timestamp} -->`,
    `<!-- roadie:updated-content-below -->`,
    newContent
  ].join('\n');
}
```

> **Why append-below instead of keep-user or three-way merge:**
> 

> - "Keep user" (Option 1) silently discards Roadie's new content — the developer never knows what Roadie wanted to update. This is a silent data loss of Roadie's improvements.
> 

> - "Three-way merge" (Option 2) risks corrupting content when auto-merging code/config.
> 

> - "Append below" ensures the developer sees BOTH their edits AND Roadie's new content. They reconcile manually. Zero data loss on either side.
> 

**This is the ONLY merge strategy. There is no Option 1 or Option 2.**

---

## Concurrent Writes & Locking

The Section Manager is the write path for all 8 file generators AND the Edit Tracker. Both callers can fire at the same time (e.g., a dependency change triggers regeneration while the developer saves a manual edit to the same file). Without coordination, the last writer silently overwrites the first — which is exactly the data-loss outcome the Append Below strategy was designed to prevent.

### Lock Protocol

`SectionManager` holds a private `Map<string, Promise<void>>` keyed by absolute file path. Every call to `writeSectionFile(filePath, sections)` chains onto the existing promise for that path, guaranteeing at most one active read-merge-write cycle per file across the entire extension host.

```tsx
class SectionManager {
  private readonly locks = new Map<string, Promise<void>>();

  async writeSectionFile(
    filePath: string,
    sections: GeneratedSection[],
  ): Promise<WriteSectionResult> {
    const abs = path.resolve(filePath);
    const prior = this.locks.get(abs) ?? Promise.resolve();

    let release!: () => void;
    const next = new Promise<void>(resolve => { release = resolve; });
    this.locks.set(abs, prior.then(() => next));

    await prior; // wait for any in-flight write on this path

    try {
      return await this.writeSectionFileLocked(abs, sections);
    } finally {
      release();
      // If we were the last waiter, clean up the map entry to avoid unbounded growth.
      if (this.locks.get(abs) === prior.then(() => next)) {
        this.locks.delete(abs);
      }
    }
  }

  private async writeSectionFileLocked(
    abs: string,
    sections: GeneratedSection[],
  ): Promise<WriteSectionResult> {
    // 1. Read the file and capture its mtime
    const statBefore = await fs.stat(abs).catch(() => null);
    const existingContent = statBefore ? await fs.readFile(abs, 'utf-8') : '';

    // 2. Compute the merged content
    const merged = this.mergeContent(existingContent, sections);

    // 3. Stat again — if the mtime changed between step 1 and step 3, the file was
    //    modified outside our lock (e.g., VS Code saved the file from another window).
    //    Abort and signal the caller to retry.
    const statAfter = statBefore ? await fs.stat(abs).catch(() => null) : null;
    if (statBefore && statAfter && statBefore.mtimeMs !== statAfter.mtimeMs) {
      return { written: false, deferred: true, reason: 'mtime_changed', mergeConflicts: [] };
    }

    // 4. Write atomically: write to a .tmp sibling, then rename.
    const tmp = `${abs}.roadie-tmp-${process.pid}`;
    await fs.writeFile(tmp, merged, { mode: 0o644 });
    await fs.rename(tmp, abs);

    return { written: true, deferred: false, contentHash: sha256(merged), mergeConflicts: [] };
  }
}
```

### Retry Policy

A caller that receives `{ written: false, reason: 'mtime_changed' }` MUST:
1. Wait ≥ 50 ms (backoff against tight loops)
2. Re-invoke `writeSectionFile()` with the **same** sections (they are idempotent)
3. Give up after 3 attempts and return `{ written: false, reason: 'mtime_contention' }`

The File Generator Manager's `runGenerationPipeline` owns this retry loop — it is the only place retry logic lives.

### Test Cases (Mandatory)

```tsx
it('handles concurrent generator + edit-tracker writes without data loss', async () => {
  const filePath = path.join(tempDir, 'test.md');
  const initial: GeneratedSection[] = [
    { id: 'deps', content: 'Roadie: Node.js, React', priority: 'required' },
  ];
  await manager.writeSectionFile(filePath, initial);

  // Fire both writers concurrently
  const generatorWrite = manager.writeSectionFile(filePath, [
    { id: 'deps', content: 'Roadie: Node.js, React, TypeScript', priority: 'required' },
  ]);
  const editTrackerWrite = manager.writeSectionFile(filePath, [
    { id: 'deps', content: 'Roadie: Node.js, React, Zustand', priority: 'required' },
  ]);

  const [r1, r2] = await Promise.all([generatorWrite, editTrackerWrite]);

  // Exactly one must have written; the other may have merged on top
  expect(r1.written || r2.written).toBe(true);

  const finalContent = await fs.readFile(filePath, 'utf-8');

  // Both payloads must be represented in the final file (Append Below semantics)
  expect(finalContent).toMatch(/TypeScript/);
  expect(finalContent).toMatch(/Zustand/);

  // File must be valid (parseable)
  const parsed = manager.parseSections(finalContent);
  expect(parsed.length).toBeGreaterThanOrEqual(1);
});

it('returns deferred result when mtime changes during merge', async () => {
  const filePath = path.join(tempDir, 'test.md');
  await fs.writeFile(filePath, '<!-- roadie:start:s --><!-- roadie:end:s -->');

  // Inject an mtime change: touch the file while the manager is merging
  const original = manager['mergeContent'];
  manager['mergeContent'] = async function(...args) {
    await fs.utimes(filePath, new Date(), new Date(Date.now() + 5000)); // Bump mtime
    return original.apply(this, args);
  };

  const result = await manager.writeSectionFile(filePath, [
    { id: 's', content: 'new', priority: 'required' },
  ]);

  expect(result.written).toBe(false);
  expect((result as { reason: string }).reason).toBe('mtime_changed');
});

it('queues concurrent writes so at most one is in flight per file', async () => {
  const filePath = path.join(tempDir, 'lock-test.md');
  const inFlight: number[] = [];
  let maxInFlight = 0;

  const originalWrite = manager['writeSectionFileLocked'].bind(manager);
  manager['writeSectionFileLocked'] = async function(...args) {
    inFlight.push(1);
    maxInFlight = Math.max(maxInFlight, inFlight.length);
    await new Promise(r => setTimeout(r, 10));
    const result = await originalWrite(...args);
    inFlight.pop();
    return result;
  };

  await Promise.all(Array.from({ length: 10 }, (_, i) =>
    manager.writeSectionFile(filePath, [
      { id: 's', content: `write ${i}`, priority: 'required' },
    ]),
  ));

  expect(maxInFlight).toBe(1); // Lock held exclusively
});
```

### Scenario 1: First Run (No Markers in Existing File)

```
Situation:
  - .github/copilot-instructions.md exists (created by developer)
  - No Roadie markers (file is 100% human)

Action:
  1. Read existing file
  2. Check if it looks like Roadie-generated (style, content)
  3. If YES: Ask developer ("Adopt this file?")
  4. If NO: Treat as human-only file
  5. Generate NEW file with different name or append sections

Result:
  Don't overwrite existing human content
```

### Scenario 2: Developer Deletes Section Markers

```jsx
Situation:
  .github/copilot-instructions.md had Roadie markers
  Developer removed the markers (intentional)

Action:
  1. Regenerate new content with markers
  2. Try to find existing markers in the file
  3. Can't find markers = treat as human-owned file
  4. Append new Roadie sections at the BOTTOM of the existing file
     (with markers, so future regenerations work normally)
  5. Log warning: "Roadie markers were removed from [file].
     New Roadie content appended at the bottom.
     Review and reorganize as needed."

Result:
  Everything stays in one file (no orphaned .roadie-new.md backups)
  Developer sees both their content and Roadie's new sections
  Developer can reorganize as they see fit
```

### Scenario 3: Developer Moves Section

```
Situation:
  Section A: "# Dependencies" was at line 10
  Developer moved it to line 50 (reordered file)

Action:
  1. Parse file, find sections by ID (not position)
  2. Sections have IDs (e.g., "dependencies")
  3. Regardless of position, we find by ID
  4. Merge and write back

Result:
  Works correctly even if sections reordered
```

### Scenario 4: Developer Partially Edits Section

```
Situation:
  Section: "# Agents"
  Roadie generated: 3 agents
  Developer added: 1 more agent
  Developer edited: 1 agent description

Action:
  1. Hash of section content changed (developer added/edited)
  2. Conflict detected
  3. Append Below: keep developer's version intact, append new Roadie content below it
     separated by <!-- roadie:merged:{timestamp} -->
  4. Log conflict + resolution

Result:
  Developer's version is preserved
  Roadie's new content is appended below (visible, not discarded)
  Developer reviews the merged section and reconciles manually
  Both versions visible in the file — zero silent data loss
```

### Scenario 5: New Section Added by Roadie

```
Situation:
  Roadie regenerates file
  Finds NEW section (e.g., "# Testing Patterns")
  Existing file doesn't have this section

Action:
  1. Section is new (not in existing file)
  2. No conflict, just add it
  3. Include markers and hash

Result:
  New section added automatically
```

### Scenario 6: Section Removed by Roadie

```
Situation:
  Old version: Copilot Instructions had "# Python Patterns"
  New version: (project is now Node.js, Python patterns removed)
  Existing file: still has "# Python Patterns" (added by developer)

Action:
  1. Regenerated content doesn't include "# Python Patterns"
  2. But existing file does (and it's in Roadie markers)
  3. Check if developer edited it
  4. If YES: keep it (user added useful content)
  5. If NO: remove it (unused Roadie section)

Result:
  Developer's additions are preserved
  Obsolete Roadie sections removed
```

---

## Section Definitions

Each generator defines sections it creates:

```tsx
interface SectionDefinition {
  id: string; // Unique ID for this section
  name: string; // Human-readable name
  description: string; // What this section contains
  fileType: 'markdown' | 'yaml' | 'json' | 'shell';
  priority: 'required' | 'recommended' | 'optional';
  }

// Example: Copilot Instructions generator
const sections: SectionDefinition[] = [
  {
    id: 'tech-stack',
    name: 'Tech Stack',
    description: 'Framework, libraries, and tools',
    fileType: 'markdown',
    priority: 'required'
  },
  {
    id: 'python-patterns',
    name: 'Python Patterns',
    description: 'Python-specific conventions (if project uses Python)',
    fileType: 'markdown',
    priority: 'optional'
  },
  // ... more sections
];
```

---

## Interface & Public API

```tsx
interface SectionManager {
  // Main operations
  writeSectionFile(
    filePath: string,
    sections: GeneratedSection[],
    existingContent?: string
  ): Promise<WriteResult>;
  
  // Parsing
  parseSections(content: string): ParsedSection[];
  
  // Merging
  mergeContent(
    existing: string,
    new: string,
    sections: SectionDefinition[]
  ): MergeResult;
  
  // Hashing
  computeHash(content: string): string;
  verifyHash(content: string, expectedHash: string): boolean;
}

interface GeneratedSection {
  id: string;
  content: string;
  priority: 'required' | 'recommended' | 'optional';
}

interface ParsedSection {
  id: string;
  startLine: number;
  endLine: number;
  content: string;
  hash: string | null; // From comment
  fileType: 'markdown' | 'yaml' | 'json' | 'shell';
}

interface WriteResult {
  success: boolean;
  filePath: string;
  written: boolean; // Did we actually write?
  changed: boolean; // Did content change?
  mergeConflicts: MergeConflict[];
  diffs: Diff[];
}

interface MergeConflict {
  sectionId: string;
  reason: 'user-edited' | 'marker-deleted' | 'content-changed';
  userVersion: string;
  newVersion: string;
  /** Resolution is always 'append-below'. 'keep-user' is not a valid option — it causes silent data loss. */
  resolution: 'append-below';
}
```

---

## Testing Strategy

### Unit Tests

```tsx
// test/generators/section-manager.test.ts

describe('Section Manager', () => {
  describe('Hash Computation', () => {
    it('computes consistent hash for same content', () => {
      const content = 'def hello():\n  print("world")';
      const hash1 = manager.computeHash(content);
      const hash2 = manager.computeHash(content);
      expect(hash1).toBe(hash2);
    });
    
    it('ignores formatting differences', () => {
      const content1 = 'name:  John';
      const content2 = 'name: John';
      // Both should hash the same
      expect(manager.computeHash(content1)).toBe(manager.computeHash(content2));
    });
  });
  
  describe('Section Parsing', () => {
    it('parses Markdown sections with markers', () => {
      const content = `
# Header
<!-- roadie:start:dependencies -->
<!-- hash:a1b2c3d4 -->
Roadie content
<!-- roadie:end:dependencies -->
`;
      const sections = manager.parseSections(content);
      expect(sections.length).toBe(1);
      expect(sections[0].id).toBe('dependencies');
      expect(sections[0].hash).toBe('a1b2c3d4');
    });
    
    it('parses YAML sections with markers', () => {
      const content = `
# roadie:start:agents
name: test
# roadie:end:agents
`;
      const sections = manager.parseSections(content);
      expect(sections.length).toBe(1);
      expect(sections[0].id).toBe('agents');
    });
  });
  
  describe('Merging', () => {
    it('keeps Roadie content if user didn\'t edit', () => {
      const existing = `
<!-- roadie:start:deps -->
<!-- hash:a1b2c3d4 -->
Old roadie content
<!-- roadie:end:deps -->
`;
      const newContent = `
<!-- roadie:start:deps -->
New roadie content
<!-- roadie:end:deps -->
`;
      
      const result = manager.mergeContent(existing, newContent, []);
      
      // Hash still matches (user didn't edit)
      expect(result.merged).toContain('New roadie content');
    });
    
    it('appends new content below user edits (append-below strategy)', () => {
      const existing = `
<!-- roadie:start:deps -->
<!-- hash:a1b2c3d4 -->
Roadie content
USER ADDED THIS LINE
<!-- roadie:end:deps -->
`;
      const newContent = `
<!-- roadie:start:deps -->
New roadie content
<!-- roadie:end:deps -->
`;
      
      const result = manager.mergeContent(existing, newContent, []);
      
      // Hash changed (user edited) — append below
      expect(result.merged).toContain('USER ADDED THIS LINE');
      expect(result.merged).toContain('New roadie content');
      expect(result.merged).toContain('roadie:merged:');
    });
    
    it('handles deleted markers (scenario 2)', () => {
      const existing = `
Roadie content (no markers)
`;
      const newContent = `
<!-- roadie:start:section -->
New roadie content
<!-- roadie:end:section -->
`;
      
      const result = manager.mergeContent(existing, newContent, []);
      
      // Can't merge (no markers in existing)
      expect(result.conflicts.length).toBeGreaterThan(0);
      expect(result.backupFile).toBeDefined(); // .roadie-new.md
    });
    
    it('handles moved sections', () => {
      const existing = `
<!-- roadie:start:deps -->
<!-- hash:a1b2c3d4 -->
Content
<!-- roadie:end:deps -->

Other stuff

<!-- roadie:start:patterns -->
<!-- hash:e5f6g7h8 -->
More content
<!-- roadie:end:patterns -->
`;
      const newContent = `
<!-- roadie:start:patterns -->
More content
<!-- roadie:end:patterns -->

<!-- roadie:start:deps -->
Content
<!-- roadie:end:deps -->
`; // Reordered
      
      const result = manager.mergeContent(existing, newContent, []);
      
      // Should work (finds sections by ID, not position)
      expect(result.success).toBe(true);
    });
  });
  
  describe('Edge Cases', () => {
    it('handles nested section IDs', () => {
      // Section ID with special chars: "python-patterns-v2"
      const content = `
<!-- roadie:start:python-patterns-v2 -->
Content
<!-- roadie:end:python-patterns-v2 -->
`;
      const sections = manager.parseSections(content);
      expect(sections[0].id).toBe('python-patterns-v2');
    });
    
    it('handles empty sections', () => {
      const content = `
<!-- roadie:start:empty -->
<!-- roadie:end:empty -->
`;
      const sections = manager.parseSections(content);
      expect(sections.length).toBe(1);
      expect(sections[0].content.trim()).toBe('');
    });
    
    it('handles malformed markers gracefully', () => {
      const content = `
<!-- roadie:start:section
Missing end marker
`;
      // Should not crash
      expect(() => manager.parseSections(content)).not.toThrow();
    });
    
    it('handles concurrent edit (file modified during generation)', async () => {
      // Simulate: Roadie reads file, developer saves, Roadie writes
      // The deferred write queue should catch this case
      const tracker = createActiveTracker();
      mockFileOpenAndUnsaved(filePath);
      
      const result = await manager.writeSectionFile(filePath, sections);
      
      // Write should be deferred, not executed
      expect(result.deferred).toBe(true);
      expect(result.written).toBe(false);
    });
    
    it('handles large files (>1MB) without timeout', async () => {
      const largeContent = generateLargeContent(1_500_000); // 1.5MB
      const sections = manager.parseSections(largeContent);
      
      // Should complete within performance budget
      const start = performance.now();
      const result = manager.mergeContent(largeContent, newContent, []);
      expect(performance.now() - start).toBeLessThan(1000);
    });
    
    it('handles files with BOM markers', () => {
      const contentWithBom = '﻿<!-- roadie:start:section -->\nContent\n<!-- roadie:end:section -->';
      const sections = manager.parseSections(contentWithBom);
      expect(sections.length).toBe(1);
    });
    
    it('detects section ID collisions between generators', () => {
      // Two generators should never produce sections with the same ID
      const allSectionDefs = getAllRegisteredSectionDefinitions();
      const ids = allSectionDefs.map(s => s.id);
      const uniqueIds = new Set(ids);
      expect(ids.length).toBe(uniqueIds.size); // No duplicates
    });
  });
});
```

### Integration Tests

```tsx
// test/integration/section-manager.integration.test.ts

describe('Section Manager Integration', () => {
  let tempDir: string;
  
  beforeEach(() => {
    tempDir = fs.mkdtempSync();
  });
  
  afterEach(() => {
    fs.rmSync(tempDir, {recursive: true});
  });
  
  it('writes file with sections on first generation', async () => {
    const filePath = path.join(tempDir, 'test.md');
    
    const sections: GeneratedSection[] = [
      {id: 'intro', content: 'Welcome', priority: 'required'}
    ];
    
    const result = await manager.writeSectionFile(filePath, sections);
    
    expect(result.success).toBe(true);
    expect(result.written).toBe(true);
    
    const content = fs.readFileSync(filePath, 'utf-8');
    expect(content).toContain('roadie:start:intro');
    expect(content).toContain('Welcome');
  });
  
  it('preserves user edits across regenerations', async () => {
    const filePath = path.join(tempDir, 'test.md');
    
    // First generation
    const section1: GeneratedSection[] = [
      {id: 'deps', content: 'Roadie: Node.js, React', priority: 'required'}
    ];
    
    await manager.writeSectionFile(filePath, section1);
    
    // User edits the file
    let content = fs.readFileSync(filePath, 'utf-8');
    content = content.replace('Roadie: Node.js, React', 'Roadie: Node.js, React\n<!-- User: Also using Webpack -->');
    fs.writeFileSync(filePath, content);
    
    // Second generation (content changed)
    const section2: GeneratedSection[] = [
      {id: 'deps', content: 'Roadie: Node.js, React, TypeScript', priority: 'required'}
    ];
    
    const result = await manager.writeSectionFile(filePath, section2, content);
    
    // Should have detected user edit and used append-below
    expect(result.mergeConflicts.length).toBeGreaterThan(0);
    expect(result.mergeConflicts[0].resolution).toBe('append-below');
    
    // User's edit AND new content should BOTH be present
    const finalContent = fs.readFileSync(filePath, 'utf-8');
    expect(finalContent).toContain('User: Also using Webpack');
    expect(finalContent).toContain('React, TypeScript');
    expect(finalContent).toContain('roadie:merged:');
  });
});
```

---

## Build Prompt for AI Agent

```
Build the Section Manager module (M22) according to this spec.

This is THE MOST CRITICAL module in Phase 1.5. Merge logic must be rock-solid.

Key requirements:
1. Parse section markers (Markdown, YAML, JSON, shell script formats)
2. Compute hashes of section content (normalized, SHA256, first 8 chars)
3. Implement merge algorithm (detect user edits, preserve them, avoid conflicts)
4. Handle 6 edge cases (first run, deleted markers, moved sections, etc.)
5. Graceful degradation: on conflict, apply the canonical "Append Below" strategy (see the "Merge Conflict Handling" section above — this is the ONLY merge strategy; never discard user edits and never discard Roadie's new content)
6. Include 40+ unit tests (parsing, merging, edge cases)
7. Include 5+ integration tests (real file system)
8. Zero silent failures (all errors logged)

Files to create:
- src/generators/section-manager.ts (main module, ~400 lines)
- src/generators/hash-utils.ts (hash computation, ~50 lines)
- src/generators/section-parser.ts (parsing logic, ~150 lines)
- src/generators/merge-algorithm.ts (merge logic, ~200 lines)
- test/generators/section-manager.test.ts (40+ tests)
- test/integration/section-manager.integration.test.ts (5+ tests)

Verification criteria:
- npm run test passes (all tests)
- npm run lint passes
- All merge scenarios tested
- All edge cases tested
- Real file system integration test passes

WARNING: This module will cause data loss if buggy (users' edits lost).
Be extremely careful. Test thoroughly. No cutting corners.
```

---

**CRITICAL DEPENDENCIES:**

1. M16: Project Model Persistence (metadata)
2. M19: File Generator Manager (knows section definitions)
3. M21: Edit Tracker (needs to know Roadie sections)

**Next Modules:**

- M23-M29: File-specific generators (use Section Manager)

**Complexity Warning:** This is the highest-risk module in Phase 1.5. Allocate extra time, review thoroughly, test exhaustively.