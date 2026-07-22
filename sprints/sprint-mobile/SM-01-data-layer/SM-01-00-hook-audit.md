# SM-01-00 — Data hook audit: verify all isDevMode guards

**Sprint:** SM-01 · Data layer  
**Branch:** `feature/sm-01-data-layer`

## Context
Several hooks guard real CF calls with `isDevMode()`. This story verifies every hook returns real data in production and that no mock path is accidentally triggered.

## Hooks to audit

| Hook | File | Real path present? | Action |
|------|------|--------------------|--------|
| `useProducts` | `hooks/useProducts.ts` | ✅ | Verify CF name matches deployed |
| `useProduct(id)` | `hooks/useProduct.ts` | ✅ | Verify |
| `useInvestments` | `hooks/useInvestments.ts` | ✅ | Verify |
| `usePortfolioStats` | `hooks/useInvestments.ts` | computed | Verify |
| `useBourseOpportunities` | `hooks/useBourse.ts` | ✅ | Verify |
| `useBourseTicker` | `hooks/useBourse.ts` | ✅ | Verify CF name |
| `useFarmerListings` | `hooks/useFinancing.ts` | ✅ | Verify |
| `useFarmerDetail` | `hooks/useFinancing.ts` | ✅ | Verify |
| `useTransactions` | `hooks/useTransactions.ts` | ❓ | Check / build if missing |
| `useNotifications` | `hooks/useLocalData.ts` | ❌ (local only) | SM-05 |

## Acceptance criteria
- [ ] All hooks above audited; file updated with correct CF function names if wrong
- [ ] `useTransactions` returns `{ data: Transaction[], isLoading }` — see SM-01-01

## Implementation notes
- CF function names live in `mombongo-functions/src/index.ts` — cross-reference
- If a CF name is wrong, the app silently returns empty data in production
