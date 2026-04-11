# 🔴 BLOCKING: Section Manager Merge Algorithm (SM-1)

**Priority:** BLOCKING — Data loss risk, merge strategy affects user experience  

**Time to Fix:** 2-3 hours  

**Risk Level:** CRITICAL (highest data loss risk in Phase 1.5)  

**Related:** Architecture Overview (already has append-below mentioned)

---

## The Problem

The Section Manager spec (M22) implements "User Priority" merge strategy. The foundation documents (PDD/TAD) and Architecture Overview specify "Append Below" merge strategy. These are DIFFERENT and have significant implications for data preservation.

---

## Comparison: Two Merge Strategies

### Current Strategy: "User Priority" ❌

When developer edits a Roadie-owned section:

```
1. Hash changed (developer edited)
2. Conflict detected
3. Resolution: KEEP user's version
4. DISCARD Roadie's new version
5. Report conflict to developer
```

**Problem:** Developer never sees what Roadie wanted to update. New Roadie content is silently lost.

### Correct Strategy: "Append Below" ✅

When developer edits a Roadie-owned section:

```
1. Hash changed (developer edited)
2. Conflict detected
3. Resolution: Keep existing content, APPEND new content below
4. Separate with: <!-- roadie:merged:{timestamp} -->
5. BOTH versions visible in file
6. Developer reviews and reconciles manually
```

**Advantage:** Developer sees both their edits AND new Roadie content. Can make informed decision about what to keep.

---

## Example: Append Below in Practice

### Scenario

Roadie generated file with section on React patterns:

```markdown
<!-- roadie:start:react-patterns -->
<!-- hash:a1b2c3d4 -->
## React Patterns
- Use React 18 hooks
- Functional components only
<!-- roadie:end:react-patterns -->
```

Developer edited it (added custom pattern):

```markdown
<!-- roadie:start:react-patterns -->
<!-- hash:a1b2c3d4 -->
## React Patterns
- Use React 18 hooks
- Functional components only
- **Custom:** Use context for theme (added by developer)
<!-- roadie:end:react-patterns -->
```

Roadie regenerates (React upgraded to 19, needs new patterns):

### With "User Priority" (❌ wrong)

Result: Developer's version is kept, Roadie's new patterns are lost

```markdown
<!-- roadie:start:react-patterns -->
## React Patterns
- Use React 18 hooks
- Functional components only
- Custom: Use context for theme (added by developer)
<!-- roadie:end:react-patterns -->
```

**Problem:** Developer doesn't know Roadie wanted to add React 19 patterns. New patterns were silently discarded.

### With "Append Below" (✅ correct)

Result: Both versions visible, developer reconciles

```markdown
<!-- roadie:start:react-patterns -->
## React Patterns
- Use React 18 hooks
- Functional components only
- Custom: Use context for theme (added by developer)
<!-- roadie:merged:2026-04-11T14:30:00Z -->
Roadie updated (2026-04-11 14:30):
## React Patterns for v19
- Use React 19 server components
- Custom hooks with lazy initialization
- Update context API usage
<!-- roadie:end:react-patterns -->
```

**Advantage:** Developer sees BOTH their content and new Roadie suggestions. Can decide to keep both, merge them, or pick selectively.

---

## Implementation Details

### Merge Algorithm with "Append Below"

```
function mergeContent(existing, new, sectionId) {
  // 1. Extract sections from both files
  const existingSection = findSection(existing, sectionId);
  const newSection = findSection(new, sectionId);
  
  // 2. Check hash
  const storedHash = getStoredHash(sectionId);
  const currentHash = computeHash(existingSection.content);
  
  if (currentHash === storedHash) {
    // No human edits, replace entirely
    return replaceSection(existing, sectionId, newSection);
  } else {
    // Human edited this section
    // Keep existing, append new below
    
    const merged = existingSection.content +
      '\n<!-- roadie:merged:' + now().toISOString() + ' -->\n' +
      'Roadie updated (' + now().toDateString() + '):\n' +
      newSection.content;
    
    return replaceSection(existing, sectionId, {
      ...newSection,
      content: merged
    });
  }
end
```

### Separator Format

```markdown
<!-- roadie:merged:2026-04-11T14:30:00Z -->
Roadie updated (2026-04-11 14:30):
```

**Key points:**

- ISO timestamp for machine parsing
- Human-readable date for developer
- Clear "Roadie updated" label

---

## Changes Required in Section Manager Spec

### 1. "Merge Algorithm" section (MAJOR REWRITE)

**Remove:**

- Option 1: User Priority
- Option 2: Three-Way Merge (rejected as "too risky")

**Add:**

- Single canonical strategy: Append Below
- Explain why this is safest (developer sees everything)
- Include exact separator format with timestamp

### 2. "Scenarios & Edge Cases" section (UPDATE)

**Scenario 4: Developer Partially Edits Section**

Old (User Priority):

```
Conflict detected
User priority: keep developer's version
Roadie's content discarded
```

New (Append Below):

```
Conflict detected
Append new Roadie content below existing
Developer sees both, can reconcile
```

### 3. "Test Cases" section (REWRITE)

**Add test for append-below merge:**

```tsx
it('appends new content below when developer edits', async () => {
  const existing = `
<!-- roadie:start:section -->
<!-- hash:a1b2c3d4 -->
Original content
Developer added this line
<!-- roadie:end:section -->
`;
  
  const new = `
<!-- roadie:start:section -->
New roadie content
<!-- roadie:end:section -->
`;
  
  const result = manager.mergeContent(existing, new, [sectionDef]);
  
  // Both versions should be visible
  expect(result).toContain('Original content');
  expect(result).toContain('Developer added this line');
  expect(result).toContain('New roadie content');
  expect(result).toContain('<!-- roadie:merged:');
});
```

---

## Why "Append Below" is Better

1. **No Silent Data Loss**
    - Developer's edits are preserved
    - Developer sees what Roadie wants to update
    - Nothing is discarded without visibility
2. **Clear Intent**
    - Timestamp shows exactly when Roadie updated
    - Developer can see the progression
    - Easy to find merge points in git history
3. **Developer Agency**
    - Developer decides what to keep
    - Not forced to accept Roadie's version
    - Not forced to lose Roadie's suggestions
4. **Audit Trail**
    - Every merge is explicitly marked
    - Can grep for `<!-- roadie:merged:` to find all merges
    - Easy to review and clean up later
5. **Backward Compatibility**
    - Works with existing files
    - Doesn't require AST parsing
    - Works with any text-based format

---

## Edge Cases Handled by Append Below

### Case 1: Developer Makes Multiple Edits Over Time

```markdown
<!-- roadie:start:section -->
Original content
Developer edit 1
<!-- roadie:merged:2026-04-10T10:00:00Z -->
Roadie update 1
Developer edit 2
<!-- roadie:merged:2026-04-11T14:30:00Z -->
Roadie update 2
<!-- roadie:end:section -->
```

Both developer and Roadie changes are visible. Easy to see history.

### Case 2: Developer Removes Old Merge Comments

Developer manually cleaned up old merge comments. New content still appended:

```markdown
<!-- roadie:start:section -->
Clean content after developer cleanup
<!-- roadie:merged:2026-04-11T14:30:00Z -->
Roadie latest update
<!-- roadie:end:section -->
```

Everything still works correctly.

### Case 3: Section Content Grows Large

Both versions accumulate in section. Developer can:

- Delete old Roadie versions (keep latest)
- Keep both if historical context is useful
- Decide based on what makes sense for their project

---

## Testing Requirements

Test cases must cover:

1. **Happy path:** No human edits, content replaced
2. **Conflict:** Hash mismatch, new content appended
3. **Multiple conflicts:** Multiple sections with conflicts
4. **Large files:** >1MB generated content
5. **Special characters:** Timestamps, Unicode in content
6. **Concurrent edits:** Developer editing while Roadie regenerating (use deferred write)
7. **Cleanup:** Developer manually removes old merge comments

---

## Timeline

**Estimated effort: 2-3 hours**

1. **Rewrite merge algorithm section (0.5 hours)**
    - Remove old options
    - Document append-below strategy
    - Add exact format examples
2. **Update scenarios & test cases (1 hour)**
    - Update all edge case scenarios
    - Add append-below test cases
    - Show example outputs
3. **Update interface & pseudocode (0.5 hours)**
    - Make sure interface matches new strategy
    - Update any helper functions
4. **Add concurrent edit handling (0.5 hours)**
    - Document deferred write behavior
    - Handle race conditions
    - Add test case

---

## Blocking This

Once this is fixed, Section Manager (M22) merge algorithm is correct and ready for implementation.

Note: Section Manager is the HIGHEST RISK module in Phase 1.5 (potential data loss). Extra scrutiny and testing required.

## Related Issues

- **SM-2:** Hash normalization behavior (document)
- **SM-3:** Backup file strategy (replace with append-to-original)
- **SM-4:** Add concurrent edit test cases
- **Architecture Overview:** Already shows append-below correctly