# ✏️ Edit Tracker Specification

## Detects Developer Modifications to Generated Files, Computes Diffs, Stores Snapshots

---

## Module Identity

**Module ID:** M21

**File Location:** `src/tracking/edit-tracker.ts`

**Depends On:** File Watcher Manager (M15), Learning Database (M23), Section Manager (M22)

**Used By:** File Generator Manager (knows when to use append-below merge)

**Complexity:** Low-Medium (diff computation, snapshot storage)

**Estimated Build Time:** 3-4 hours

**Implementation Status:** ✅ COMPLETE — Implemented as of 2026-04-12

---

## Responsibility

Detect when developers modify Roadie-generated files (`.github/**`) and record the modifications for the learning system. This module answers: "What did the developer change, and when?"

---

## How It Works

1. File Watcher classifies a change to `.github/copilot-instructions.md` as `USER_EDIT`
2. Edit Tracker receives the event
3. Edit Tracker retrieves the last known snapshot from Learning Database
4. Edit Tracker reads the current file content
5. Edit Tracker computes a diff (what changed)
6. Edit Tracker stores a new snapshot with `source: 'human'`
7. Edit Tracker determines which Roadie-owned sections were modified (via Section Manager)

---

## Public Interface

```tsx
interface EditTracker {
  /** Process a USER_EDIT event from the file watcher */
  trackEdit(filePath: string): Promise<EditRecord | null>;
  
  /** Check if a file has been edited by the developer since last generation */
  hasHumanEdits(filePath: string): Promise<boolean>;
  
  /** Get edit history for a specific file */
  getEditHistory(filePath: string, limit?: number): Promise<EditRecord[]>;
  
  /** Initialize: only activate if editTracking config is enabled */
  initialize(config: RoadieConfig): void;
  
  /** Dispose: cleanup subscriptions */
  dispose(): void;
}

interface EditRecord {
  filePath: string;
  timestamp: Date;
  editedSections: string[];   // IDs of Roadie sections that were modified
  addedOutsideMarkers: boolean; // Developer added content outside Roadie markers
  diffSummary: {
    linesAdded: number;
    linesRemoved: number;
    linesModified: number;
  };
}
```

---

## Activation Guard

Edit tracking is **opt-in** (default: `false`):

```tsx
initialize(config: RoadieConfig): void {
  if (!config.editTracking) {
    logger.info('Edit tracking disabled (opt-in via roadie.editTracking)');
    this.active = false;
    return;
  }
  this.active = true;
  // Subscribe to USER_EDIT events from file watcher
}
```

When disabled, `trackEdit()` returns `null` immediately. `hasHumanEdits()` falls back to hash comparison via Section Manager (works without stored history).

---

## Diff Computation

```tsx
async function trackEdit(filePath: string): Promise<EditRecord | null> {
  if (!this.active) return null;
  
  // 1. Get last snapshot
  const lastSnapshot = await this.learningDb.getSnapshots(filePath, 1);
  const previousContent = lastSnapshot[0]?.content ?? '';
  
  // 2. Read current content
  const currentContent = await fs.promises.readFile(filePath, 'utf-8');
  
  // 3. If identical, skip
  if (computeHash(previousContent) === computeHash(currentContent)) {
    return null;
  }
  
  // 4. Compute diff summary
  const diffSummary = computeDiffSummary(previousContent, currentContent);
  
  // 5. Determine which Roadie sections were edited
  const previousSections = sectionManager.parseSections(previousContent);
  const currentSections = sectionManager.parseSections(currentContent);
  const editedSections = findEditedSections(previousSections, currentSections);
  
  // 6. Check if content was added outside markers
  const addedOutsideMarkers = hasContentOutsideMarkers(
    previousContent, currentContent, currentSections
  );
  
  // 7. Store snapshot
  await this.learningDb.recordSnapshot(filePath, currentContent, 'human');
  
  const record: EditRecord = {
    filePath,
    timestamp: new Date(),
    editedSections: editedSections.map(s => s.id),
    addedOutsideMarkers,
    diffSummary
  };
  
  logger.info(
    `Edit tracked: ${filePath} — ${diffSummary.linesModified} lines modified, ` +
    `${editedSections.length} Roadie sections edited`
  );
  
  return record;
}

function computeDiffSummary(oldContent: string, newContent: string): DiffSummary {
  const oldLines = oldContent.split('\n');
  const newLines = newContent.split('\n');
  
  // Simple line-by-line comparison
  let added = 0, removed = 0, modified = 0;
  const maxLen = Math.max(oldLines.length, newLines.length);
  
  for (let i = 0; i < maxLen; i++) {
    if (i >= oldLines.length) { added++; continue; }
    if (i >= newLines.length) { removed++; continue; }
    if (oldLines[i] !== newLines[i]) { modified++; }
  }
  
  return { linesAdded: added, linesRemoved: removed, linesModified: modified };
}
```

---

## Testing Strategy

```tsx
describe('EditTracker', () => {
  it('returns null when edit tracking is disabled', async () => {
    const tracker = new EditTracker();
    tracker.initialize({ editTracking: false } as RoadieConfig);
    
    const result = await tracker.trackEdit('.github/copilot-instructions.md');
    expect(result).toBeNull();
  });
  
  it('detects edits inside Roadie sections', async () => {
    const tracker = createActiveTracker();
    // Store initial snapshot
    await learningDb.recordSnapshot(filePath, originalContent, 'roadie');
    // Modify file
    await writeFile(filePath, editedContent);
    
    const result = await tracker.trackEdit(filePath);
    
    expect(result).not.toBeNull();
    expect(result!.editedSections).toContain('tech-stack');
  });
  
  it('detects content added outside markers', async () => {
    const tracker = createActiveTracker();
    await learningDb.recordSnapshot(filePath, originalContent, 'roadie');
    // Add content outside markers
    const edited = originalContent + '\n## My Custom Section\nMy notes here';
    await writeFile(filePath, edited);
    
    const result = await tracker.trackEdit(filePath);
    
    expect(result!.addedOutsideMarkers).toBe(true);
  });
  
  it('skips when content is unchanged', async () => {
    const tracker = createActiveTracker();
    await learningDb.recordSnapshot(filePath, content, 'roadie');
    
    const result = await tracker.trackEdit(filePath);
    expect(result).toBeNull();
  });
  
  it('stores snapshot with source human', async () => {
    const tracker = createActiveTracker();
    await learningDb.recordSnapshot(filePath, originalContent, 'roadie');
    await writeFile(filePath, editedContent);
    
    await tracker.trackEdit(filePath);
    
    const snapshots = await learningDb.getSnapshots(filePath, 1);
    expect(snapshots[0].source).toBe('human');
  });
});
```

---

## Build Prompt for AI Agent

```
Build the Edit Tracker module (M21) according to this spec.

Key requirements:
1. Only active when roadie.editTracking is true (opt-in, default false)
2. Detect edits to .github/ files via USER_EDIT events from file watcher
3. Compare current file against last snapshot from Learning Database
4. Compute diff summary (lines added/removed/modified)
5. Identify which Roadie-owned sections were edited (via Section Manager)
6. Store new snapshot with source 'human'
7. All file I/O is async
8. Include 10+ unit tests

Files to create:
- src/tracking/edit-tracker.ts (~150 lines)
- src/tracking/edit-tracker.test.ts (~200 lines)
```