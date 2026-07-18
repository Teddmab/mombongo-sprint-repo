# SA-07 — Academia Admin: Polish + Module Reorder + Sync

## Goal
Fix the stubbed module reorder buttons, add module edit, and keep `moduleCount` on the course document in sync when modules are added or deleted.

## Current state
- Module ChevronUp/ChevronDown onClick handlers contain empty comments — do nothing
- No module edit (update title/type/duration/isFree) after creation
- `moduleCount` on the course doc is set at course creation and never updated when modules are added/deleted

## Work items

### 1. Module reorder
- On ChevronUp: swap `order` fields between module[i] and module[i-1] — batch write
- On ChevronDown: swap `order` fields between module[i] and module[i+1] — batch write
- Use `writeBatch` from Firestore: update both docs atomically

### 2. Module edit
- Add pencil icon to module rows (currently only delete exists)
- Open a `ModuleEditForm` modal (pre-fill title, type, durationMinutes, isFree, contentUrl)
- `updateDoc(courses/{courseId}/modules/{moduleId}, changes)`

### 3. Sync moduleCount
- After `addDoc` a module: `updateDoc(courses/{courseId}, { moduleCount: increment(1) })`
- After `deleteDoc` a module: `updateDoc(courses/{courseId}, { moduleCount: increment(-1) })`
- Use `increment` from `firebase/firestore`

### 4. Module content URL field
- Add `contentUrl` field to the ModuleForm (video YouTube URL, PDF URL, or quiz JSON)
- Display truncated URL in the module row

## Acceptance criteria
- [ ] ChevronUp/Down reorder modules — order persists after page refresh
- [ ] Module edit saves changes to Firestore
- [ ] Adding a module increments course moduleCount
- [ ] Deleting a module decrements course moduleCount
