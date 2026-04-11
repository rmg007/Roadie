# 🔴 BLOCKING: File Watcher API Restructuring (FW-1)

**Priority:** BLOCKING — Cannot build File Watcher Manager without this fix  

**Time to Fix:** 4-5 hours  

**Related:** C1 (chokidar vs VS Code FileSystemWatcher)

---

## The Problem

The File Watcher Manager spec (M15) was written for chokidar API but the foundation documents specify VS Code's FileSystemWatcher.

**These are DIFFERENT APIs with incompatible event models:**

### Chokidar API (what the spec currently uses)

```jsx
watcher.on('add', (filePath) => { /* new file */ })
watcher.on('change', (filePath) => { /* modified file */ })
watcher.on('unlink', (filePath) => { /* deleted file */ })
watcher.on('addDir', (dirPath) => { /* new directory */ })  ← NO EQUIVALENT IN VS CODE
watcher.on('unlinkDir', (dirPath) => { /* deleted directory */ })  ← NO EQUIVALENT IN VS CODE
```

### VS Code FileSystemWatcher API (what must be used)

```jsx
const watcher = vscode.workspace.createFileSystemWatcher(pattern);

watcher.onDidCreate(uri => { /* new file */ })
watcher.onDidChange(uri => { /* modified file */ })
watcher.onDidDelete(uri => { /* deleted file */ })
// NO DIRECTORY EVENTS — must infer from file paths
```

---

## Impact on File Watcher Spec

### Changes Required

1. **Remove directory event handling**
    - Cannot detect directory creation with FileSystemWatcher
    - Cannot detect directory deletion directly
    - Must infer STRUCTURE_CHANGE from file paths instead
2. **Change event classification**
    - If file created in new path: infer STRUCTURE_CHANGE
    - If all files in directory deleted: infer directory removal
    - Watch project model's directory tree to track changes
3. **Debouncing**
    - Chokidar has no built-in debouncing
    - VS Code FileSystemWatcher also has no debouncing
    - Must implement custom debouncing (500ms, same as spec)
    - Already specified correctly in spec
4. **Workspace Trust Integration**
    - VS Code FileSystemWatcher respects workspace trust
    - Must skip watching in untrusted workspaces
    - Chokidar doesn't have this concept
5. **Remote Development**
    - VS Code FileSystemWatcher works in Remote Development (SSH, WSL, etc.)
    - Chokidar doesn't work reliably in Remote Dev
    - This is a significant advantage of VS Code API

### Code Changes Needed

**Old (chokidar-based):**

```
watcher.on('addDir', (dirPath) => {
  return {
    type: 'STRUCTURE_CHANGE',
    triggers: ['path-instructions']
  };
});

watcher.on('unlinkDir', (dirPath) => {
  // Handle directory deletion
});
```

**New (VS Code API-based):**

```
const watcher = vscode.workspace.createFileSystemWatcher(watchPatterns);

watcher.onDidCreate(uri => {
  // Check if file path suggests structure change
  // e.g., first file in new directory → infer STRUCTURE_CHANGE
  if (isFirstFileInDirectory(uri)) {
    return {
      type: 'STRUCTURE_CHANGE',
      triggers: ['path-instructions']
    };
  }
});

watcher.onDidDelete(uri => {
  // Check project model's directory tree
  // If all files in directory deleted → infer directory removal
  if (isLastFileInDirectory(uri)) {
    return {
      type: 'STRUCTURE_CHANGE',
      triggers: ['path-instructions']
    };
  }
});
```

---

## Sections That Need Complete Rewrite

In the File Watcher Manager Specification (M15):

1. **"Change Classification" section**
    - Remove reference to addDir/unlinkDir events
    - Add logic for inferring structure changes from file paths
    - Show how to check project model tree
2. **"Chokidar Events" section (header + content)**
    - Rename to "VS Code FileSystemWatcher Events"
    - Change from `watcher.on()` to `watcher.onDidCreate/Change/Delete`
    - Update event properties (uri instead of filePath)
3. **"Layer 1: File System Monitoring" in Architecture Overview**
    - Already corrected (shows FileSystemWatcher, not chokidar)
    - But File Watcher spec still needs alignment
4. **Error Handling section**
    - VS Code FileSystemWatcher has different error model
    - Workspace trust errors vs permission errors
    - Fallback strategy needs update
5. **Polling Fallback section**
    - Use `vscode.workspace.findFiles()` instead of file system scan
    - Different API, same concept
6. **Performance budgets section**
    - Same budgets apply (500ms debounce, 1s latency)
    - FileSystemWatcher actually more efficient than chokidar

---

## Test Cases That Need Update

### Old (chokidar-based)

```tsx
watcher.on('addDir', (filePath) => {
  // Test directory event
});
```

### New (VS Code API-based)

```tsx
watcher.onDidCreate(uri => {
  // Check if first file in directory
  // Infer directory creation
});
```

---

## Workspace Trust Considerations

VS Code FileSystemWatcher respects workspace trust automatically:

```tsx
if (vscode.workspace.isTrusted === false) {
  logger.warn('Untrusted workspace: file watcher disabled');
  // Skip watching, use polling or disable Phase 1.5
}
```

Chokidar doesn't have this concept — it's an additional consideration.

---

## Timeline

**Estimated effort: 4-5 hours**

1. **Rewrite change classification logic (1 hour)**
    - Remove directory event handling
    - Add path-based inference
    - Update pseudocode
2. **Rewrite event handling section (1.5 hours)**
    - Change from chokidar to VS Code API
    - Update all event handler code
    - Update properties (filePath → uri)
3. **Update error handling (1 hour)**
    - Adjust for VS Code API error model
    - Add workspace trust handling
    - Update recovery strategies
4. **Update/add tests (1-1.5 hours)**
    - Rewrite all event-based tests
    - Add workspace trust tests
    - Add remote development tests

---

## What's NOT Changing

- ✅ Debounce strategy (500ms, same)
- ✅ Classification types (DEPENDENCY_CHANGE, STRUCTURE_CHANGE, etc., same)
- ✅ Performance budgets (same)
- ✅ Dispatch routing (same)
- ✅ Retry/escalation logic (same)

---

## Blocking This

Once this is fixed, File Watcher Manager (M15) spec will be ready for AI agent implementation.

**Next:** Project Model Persistence (M14) spec also blocking.

## Related Issues

- **C1:** Chokidar vs FileSystemWatcher (foundation document contradiction)
- **FW-4:** Startup reconciliation (how to catch files changed while VS Code was closed)
- **Architecture Overview:** Already partially corrected, but M15 spec needs full rewrite