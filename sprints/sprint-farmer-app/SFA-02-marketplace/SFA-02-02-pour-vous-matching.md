# SFA-02-02 — "Pour vous" Buyer Matching (Farmer App)

## Context
Port of web sprint SU-03-02. When a farmer has a listing on the Bourse, the `getBourseBuyerMatches` CF returns buyers whose demand overlaps with the farmer's commodity, province, and quantity. This tab shows "buyers looking for your crop" to motivate the farmer to keep their listing active.

## Scope
- Create `src/hooks/useBourseBuyerMatches.ts` — calls `getBourseBuyerMatches` CF
- Add a "Pour vous" tab or section in `FarmerBourseScreen.tsx`
- Each match card: buyer province, desired quantity, offered price range, contact button

## Cloud Function required
`getBourseBuyerMatches` — planned in SU-03-02. Input: void (reads farmer's cropType/province from user profile). Output:
```typescript
{
  matches: Array<{
    buyerProvince: string
    desiredQuantityKg: number
    offerPricePerKgCdf: number
    matchScore: number // 0-100
  }>
}
```

## Files to create / modify
- `src/hooks/useBourseBuyerMatches.ts`
- `src/screens/bourse/FarmerBourseScreen.tsx` — add "Pour vous" section below listings

## Implementation

### `src/hooks/useBourseBuyerMatches.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

const MOCK_MATCHES = [
  { buyerProvince: 'Kinshasa', desiredQuantityKg: 2000, offerPricePerKgCdf: 460, matchScore: 92 },
  { buyerProvince: 'Kongo Central', desiredQuantityKg: 500, offerPricePerKgCdf: 440, matchScore: 75 },
]

export function useBourseBuyerMatches() {
  return useQuery({
    queryKey: ['buyerMatches'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_MATCHES
      const res = await httpsCallable<void, { matches: typeof MOCK_MATCHES }>(
        functions, 'getBourseBuyerMatches'
      )()
      return res.data.matches
    },
    staleTime: 15 * 60_000,
  })
}
```

### FarmerBourseScreen — "Pour vous" section
```typescript
// Below listings list, above FAB:
<SectionHeader title="Acheteurs pour vous" />
{matches?.map(m => (
  <BuyerMatchCard key={`${m.buyerProvince}-${m.desiredQuantityKg}`} match={m} />
))}
```

### BuyerMatchCard
```typescript
// Province, quantity wanted, offered price, match score badge
// "Contacter" button → navigate to messaging or contact modal (future)
```

## Acceptance criteria
- [ ] "Pour vous" section appears in Bourse tab when matches exist
- [ ] Empty state shown when no matches ("Aucun acheteur pour votre récolte pour l'instant")
- [ ] Each card shows province, quantity, price, and match score
- [ ] Dev mode shows 2 mock matches

## Smoke test
1. Open Bourse tab — scroll to "Pour vous" section
2. Verify 2 mock buyer match cards appear in dev mode
3. In live mode: verify real matches load for a farmer with `cropType` set
