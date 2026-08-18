# SFA-02-01 — Bourse Publish + Progress Stepper (Farmer App)

## Context
Port of web sprint SU-02-04. The Bourse farmer side (publish listing, track buyer interest) exists in mobile as `FarmerBourseScreen.tsx` but is mostly a skeleton. This sprint wires it to real CFs and adds the 5-stage progress stepper showing where the farmer is in the selling journey.

5 stages (same as web):
1. **Publié** — listing is live
2. **Intérêt reçu** — at least one buyer contact
3. **Négociation** — price agreed, awaiting confirmation
4. **Contrat signé** — escrow created
5. **Livré & payé** — delivery confirmed, payout released

## Scope
- Wire `FarmerBourseScreen.tsx` to real `getMyListings` CF
- Create `src/components/BourseStepperCard.tsx` — horizontal 5-stage stepper
- Create `CreateListingSheet.tsx` — bottom sheet to publish a new listing
- Create `src/hooks/useMyBourseListings.ts`

## Cloud Functions required
- `getMyListings` — farmer's own listings (already in bourse CFs from S3/S8)
- `createProductListing` — publish a listing (already exists from S8)

## Files to create / modify
- `src/hooks/useMyBourseListings.ts`
- `src/screens/bourse/FarmerBourseScreen.tsx` — wire + add stepper
- `src/components/BourseStepperCard.tsx`
- `src/screens/bourse/CreateListingSheet.tsx`

## Implementation

### `src/hooks/useMyBourseListings.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

const MOCK_LISTINGS = [
  { id: '1', crop: 'Maïs', quantityKg: 500, pricePerKgCdf: 450, status: 'negotiating', buyerCount: 3 },
]

export function useMyBourseListings() {
  return useQuery({
    queryKey: ['myBourseListings'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_LISTINGS
      const res = await httpsCallable<void, { listings: typeof MOCK_LISTINGS }>(
        functions, 'getMyListings'
      )()
      return res.data.listings
    },
  })
}
```

### BourseStepperCard.tsx
```typescript
const STAGES = ['Publié', 'Intérêt reçu', 'Négociation', 'Contrat signé', 'Livré & payé']
const STATUS_TO_STAGE: Record<string, number> = {
  published: 0,
  interested: 1,
  negotiating: 2,
  contracted: 3,
  delivered: 4,
}

export function BourseStepperCard({ listing }: { listing: Listing }) {
  const currentStage = STATUS_TO_STAGE[listing.status] ?? 0
  return (
    <View style={styles.card}>
      <Text style={styles.crop}>{listing.crop} · {listing.quantityKg}kg</Text>
      <View style={styles.stepper}>
        {STAGES.map((stage, i) => (
          <StageNode key={stage} label={stage} index={i} current={currentStage} />
        ))}
      </View>
      {listing.buyerCount > 0 && (
        <Text style={styles.interest}>{listing.buyerCount} acheteur(s) intéressé(s)</Text>
      )}
    </View>
  )
}
```

## Acceptance criteria
- [ ] FarmerBourseScreen shows farmer's listings from real CF
- [ ] Each listing card shows the 5-stage stepper with correct current stage highlighted
- [ ] FAB opens CreateListingSheet
- [ ] CreateListingSheet submits to `createProductListing` CF
- [ ] After creation, listing appears in the list
- [ ] Buyer count shown when > 0

## Smoke test
1. Open Bourse tab as farmer — confirm listings load from CF
2. Verify stepper stage is correct for each listing status
3. Tap FAB → fill listing form → submit → confirm listing appears in list
4. Open Firebase console → verify listing document created
