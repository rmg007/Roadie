# 📚 Learning Database Specification

## SQLite Storage for Edit History, Workflow Outcomes, Discovered Patterns, Pruning

---

## Module Identity

**Module ID:** M23

**File Location:** `src/learning/learning-database.ts`

**Depends On:** Database module (M5 — shared SQLite connection)

**Used By:** Edit Tracker, File Generator Manager, Workflow Engine (outcome logging)

**Complexity:** Low-Medium (CRUD + pruning, well-defined schema)

**Estimated Build Time:** 3-4 hours

**Implementation Status:** ✅ COMPLETE — Implemented as of 2026-04-12

---

## Critical Design Decision: Single Database

All data lives in ONE SQLite database: `.github/.roadie/project-model.db`

The Learning Database shares the same SQLite connection as the Project Model. Do NOT create a separate `learning.db` file. This simplifies transactions, backup/restore, and corruption recovery.

---

## SQLite Schema (Additions to Project Model DB)

```sql
-- File snapshots: full content of generated files after each generation or human edit
CREATE TABLE IF NOT EXISTS file_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path TEXT NOT NULL,
  content TEXT NOT NULL,
  content_hash TEXT NOT NULL,    -- SHA-256 of content
  source TEXT NOT NULL,          -- 'roadie' or 'human'
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_snapshots_path 
  ON file_snapshots(file_path, created_at DESC);

-- Workflow history: outcomes of workflow executions
CREATE TABLE IF NOT EXISTS workflow_history (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  workflow_type TEXT NOT NULL,    -- 'bug_fix', 'feature', etc.
  prompt TEXT NOT NULL,           -- developer's original prompt
  status TEXT NOT NULL,           -- 'completed', 'failed', 'cancelled'
  steps_completed INTEGER NOT NULL DEFAULT 0,
  steps_total INTEGER NOT NULL DEFAULT 0,
  duration_ms INTEGER,
  model_tiers_used TEXT,          -- JSON array: ['free', 'standard']
  error_summary TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Section hashes: tracks last-generated content hash per section
CREATE TABLE IF NOT EXISTS section_hashes (
  file_path TEXT NOT NULL,
  section_id TEXT NOT NULL,
  content_hash TEXT NOT NULL,     -- SHA-256 of last Roadie-generated content
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (file_path, section_id)
);
```

---

## Public Interface

```tsx
interface LearningDatabase {
  // === File Snapshots ===
  recordSnapshot(filePath: string, content: string, source: 'roadie' | 'human'): Promise<void>;
  getSnapshots(filePath: string, limit?: number): Promise<FileSnapshot[]>;
  getLatestSnapshot(filePath: string): Promise<FileSnapshot | null>;
  
  // === Workflow History ===
  recordWorkflowOutcome(entry: WorkflowOutcomeInput): Promise<void>;
  getWorkflowHistory(limit?: number): Promise<WorkflowHistoryEntry[]>;
  getWorkflowStats(): Promise<WorkflowStats>;
  
  // === Section Hashes ===
  getSectionHash(filePath: string, sectionId: string): Promise<string | null>;
  setSectionHash(filePath: string, sectionId: string, hash: string): Promise<void>;
  
  // === Maintenance ===
  prune(): Promise<PruneResult>;
  getDatabaseSize(): Promise<number>;  // bytes
  
  // === Lifecycle ===
  initialize(db: BetterSqlite3.Database): void;
  close(): void;
}

interface WorkflowOutcomeInput {
  workflowType: string;
  prompt: string;
  status: 'completed' | 'failed' | 'cancelled';
  stepsCompleted: number;
  stepsTotal: number;
  durationMs: number;
  modelTiersUsed: string[];
  errorSummary?: string;
}

interface WorkflowStats {
  totalWorkflows: number;
  completionRate: number;       // 0.0-1.0
  averageDuration: number;      // ms
  tierDistribution: { free: number; standard: number; premium: number };
}

interface PruneResult {
  snapshotsRemoved: number;
  historyEntriesRemoved: number;
  bytesFreed: number;
}
```

---

## Activation Guard

Workflow history and edit tracking are **opt-in** (defaults: `false`):

```tsx
async recordWorkflowOutcome(entry: WorkflowOutcomeInput): Promise<void> {
  if (!this.config.workflowHistory) {
    return; // Silently skip
  }
  // ... store in SQLite
}

// File snapshots are ALWAYS recorded (needed for Section Manager hash comparison)
// regardless of editTracking config. The editTracking flag controls whether
// the Edit Tracker actively monitors for human edits, not whether snapshots are stored.
```

---

## Retention & Pruning

```tsx
async prune(): Promise<PruneResult> {
  let snapshotsRemoved = 0;
  let historyRemoved = 0;
  
  this.db.transaction(() => {
    // Keep last 50 snapshots per file
    const files = this.db.prepare(
      'SELECT DISTINCT file_path FROM file_snapshots'
    ).all();
    
    for (const { file_path } of files) {
      const deleted = this.db.prepare(
        `DELETE FROM file_snapshots 
         WHERE file_path = ? 
         AND id NOT IN (
           SELECT id FROM file_snapshots 
           WHERE file_path = ? 
           ORDER BY created_at DESC 
           LIMIT 50
         )`
      ).run(file_path, file_path);
      snapshotsRemoved += deleted.changes;
    }
    
    // Keep last 100 workflow history entries
    const historyDeleted = this.db.prepare(
      `DELETE FROM workflow_history 
       WHERE id NOT IN (
         SELECT id FROM workflow_history 
         ORDER BY created_at DESC 
         LIMIT 100
       )`
    ).run();
    historyRemoved = historyDeleted.changes;
  })();
  
  // VACUUM to reclaim space
  this.db.exec('VACUUM');
  
  return {
    snapshotsRemoved,
    historyEntriesRemoved: historyRemoved,
    bytesFreed: 0 // Approximation not worth computing
  };
}
```

Pruning runs on extension activation (not during workflows).

---

## Size Budget

Target: database stays under **10 MB** for typical projects.

With retention limits:

- 50 snapshots × 20 files × ~5KB average = ~5 MB
- 100 workflow history entries × ~1KB = ~100 KB
- Section hashes: negligible (<10 KB)
- Detected patterns: negligible (<50 KB)
- **Total: ~5.2 MB** (well within budget)

If `getDatabaseSize()` exceeds 10 MB, log a warning. Do not auto-delete — let pruning handle it on next activation.

---

## Testing Strategy

```tsx
describe('LearningDatabase', () => {
  let db: LearningDatabase;
  
  beforeEach(() => {
    db = createTestLearningDb(); // In-memory SQLite
  });
  
  describe('File Snapshots', () => {
    it('records and retrieves snapshots', async () => {
      await db.recordSnapshot('test.md', 'content', 'roadie');
      const snapshots = await db.getSnapshots('test.md');
      expect(snapshots).toHaveLength(1);
      expect(snapshots[0].source).toBe('roadie');
    });
    
    it('orders by most recent first', async () => {
      await db.recordSnapshot('test.md', 'v1', 'roadie');
      await db.recordSnapshot('test.md', 'v2', 'human');
      const snapshots = await db.getSnapshots('test.md');
      expect(snapshots[0].content).toBe('v2');
    });
    
    it('respects limit parameter', async () => {
      for (let i = 0; i < 10; i++) {
        await db.recordSnapshot('test.md', `v${i}`, 'roadie');
      }
      const snapshots = await db.getSnapshots('test.md', 3);
      expect(snapshots).toHaveLength(3);
    });
  });
  
  describe('Workflow History', () => {
    it('records workflow outcome', async () => {
      await db.recordWorkflowOutcome({
        workflowType: 'bug_fix',
        prompt: 'fix the login bug',
        status: 'completed',
        stepsCompleted: 8,
        stepsTotal: 8,
        durationMs: 45000,
        modelTiersUsed: ['free', 'standard']
      });
      const history = await db.getWorkflowHistory();
      expect(history).toHaveLength(1);
    });
    
    it('skips recording when workflowHistory disabled', async () => {
      db.initialize(testDb, { workflowHistory: false } as RoadieConfig);
      await db.recordWorkflowOutcome(outcome);
      const history = await db.getWorkflowHistory();
      expect(history).toHaveLength(0);
    });
    
    it('computes workflow stats', async () => {
      await db.recordWorkflowOutcome({ ...outcome, status: 'completed' });
      await db.recordWorkflowOutcome({ ...outcome, status: 'completed' });
      await db.recordWorkflowOutcome({ ...outcome, status: 'failed' });
      
      const stats = await db.getWorkflowStats();
      expect(stats.totalWorkflows).toBe(3);
      expect(stats.completionRate).toBeCloseTo(0.667, 2);
    });
  });
  
  describe('Pruning', () => {
    it('keeps last 50 snapshots per file', async () => {
      for (let i = 0; i < 60; i++) {
        await db.recordSnapshot('test.md', `v${i}`, 'roadie');
      }
      
      const result = await db.prune();
      
      expect(result.snapshotsRemoved).toBe(10);
      const remaining = await db.getSnapshots('test.md');
      expect(remaining).toHaveLength(50);
    });
    
    it('keeps last 100 workflow history entries', async () => {
      for (let i = 0; i < 120; i++) {
        await db.recordWorkflowOutcome({ ...outcome, prompt: `prompt ${i}` });
      }
      
      const result = await db.prune();
      
      expect(result.historyEntriesRemoved).toBe(20);
    });
  });
  
  describe('Section Hashes', () => {
    it('stores and retrieves section hashes', async () => {
      await db.setSectionHash('test.md', 'tech-stack', 'abc123');
      const hash = await db.getSectionHash('test.md', 'tech-stack');
      expect(hash).toBe('abc123');
    });
    
    it('returns null for unknown sections', async () => {
      const hash = await db.getSectionHash('test.md', 'unknown');
      expect(hash).toBeNull();
    });
  });
});
```

---

## Build Prompt for AI Agent

```
Build the Learning Database module (M23) according to this spec.

Key requirements:
1. Uses the SAME SQLite database as Project Model (project-model.db)
2. Creates tables if they don't exist (file_snapshots, workflow_history, section_hashes)
3. File snapshots always recorded (needed by Section Manager)
4. Workflow history only recorded when config.workflowHistory is true
5. Pruning: last 50 snapshots per file, last 100 workflow entries
6. Pruning runs on activation, not during workflows
7. Database size target: < 10 MB
8. Include 15+ unit tests covering all CRUD + pruning

Files to create:
- src/learning/learning-database.ts (~200 lines)
- src/learning/learning-database.test.ts (~250 lines)
```