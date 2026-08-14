# SP-08 — Agent Market Category Crash Fix

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-08 |
| Sprint | Sprint Patch 08 |
| Branch | `feature/sp-08-fix-category-undeclared` |
| Merges into | `main` |
| Estimate | 30m |
| Status | DONE — PR #78 merged |

## Root cause
`react-dom.production.min.js:188 ReferenceError: category is not defined` thrown when agent terrain navigated to `/market`.

`PublierPourAgriculteurModal` in `src/components/forms/ActionForms.tsx` used `category` and `setCategory` in JSX (the `<select>` element and the `httpsCallable` payload) but never declared them as component state.

## Fix
Added the missing `useState` declaration alongside the other state variables:

```typescript
const [farmerId, setFarmerId] = useState(initialFarmerId ?? "");
const [name, setName] = useState("");
const [category, setCategory] = useState("agriculture");  // ← added
const [qty, setQty] = useState("");
```

Also added `category` to:
- The `httpsCallable(functions, 'publishListingForFarmer')` payload
- The form reset after successful submit

## Files changed
- `src/components/forms/ActionForms.tsx` — add `useState` for `category`, pass to CF payload and reset

## Acceptance criteria
- [x] Agent terrain: navigate to Market → no ReferenceError
- [x] Open "Publier pour un agriculteur" modal → no crash
- [x] Submit modal → `category` field included in CF payload
- [x] `npx tsc --noEmit` passes with 0 errors
