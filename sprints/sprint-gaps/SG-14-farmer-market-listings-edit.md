# SG-14 — Farmer Market: Own Listings & Edit Listing

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-14 |
| Sprint | Sprint Gaps 14 |
| Branch | `feature/sg-14-farmer-market-listings-edit` |
| Merges into | `dev` |
| Estimate | 4h |
| Dependencies | S2-01 (product listings created by admin), S8-01 (product_listings CF) |

---

## Context

`AgricultorMarket.tsx` (the Marché tab for farmers) shows:
- `myListings` — farmer's own product listings — from `mock.ts`
- A bourseTicker strip — from `mock.ts`
- "Modifier" button on each listing → toast stub "Modification bientôt disponible"

The farmer-facing marketplace (products from the platform, not the bourse) was built in Sprint 2
for investors and merchants. Farmers were never wired to see or edit their OWN listings from this
marketplace (as distinct from Bourse listings from S8).

**Clarification on two listing types**:
- **`product_listings`** (S8-01 AgroExchange, Bourse tab): farmer's own produce offered P2P
- **`products`** (S2, Marché tab): platform-curated agricultural investment products

`AgricultorMarket` shows `product_listings` (the farmer's own bourse listings) in the "Mes annonces"
section, because a farmer needs to see their active listings from one central place. The `useMyListings()`
hook from S8 was built for the Bourse tab — `AgricultorMarket` should reuse it.

---

## Cloud Function: `updateProductListing`
Allows a farmer to edit their own active listing (price, quantity, dates, description):
```typescript
// Params: { listingId, updates: { pricePerKgCdf?, quantityKg?, availableUntil?, description? } }
export const updateProductListing = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { listingId, updates } = data
  const listingRef = db.collection('product_listings').doc(listingId)
  const snap = await listingRef.get()
  if (!snap.exists) throw new functions.https.HttpsError('not-found', 'Annonce introuvable')
  if (snap.data()!.sellerId !== uid)
    throw new functions.https.HttpsError('permission-denied', 'Vous ne pouvez modifier que vos propres annonces')

  const ALLOWED = ['pricePerKgCdf', 'quantityKg', 'availableUntil', 'description', 'quality']
  const safe = Object.fromEntries(
    Object.entries(updates).filter(([k]) => ALLOWED.includes(k))
  )

  await listingRef.update({ ...safe, updatedAt: FieldValue.serverTimestamp() })
  return { ok: true }
})
```

### `deactivateProductListing`
Let a farmer close a listing (mark as 'inactive'):
```typescript
// Params: { listingId }
export const deactivateProductListing = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const ref = db.collection('product_listings').doc(data.listingId)
  const snap = await ref.get()
  if (!snap.exists) throw new functions.https.HttpsError('not-found', 'Annonce introuvable')
  if (snap.data()!.sellerId !== uid)
    throw new functions.https.HttpsError('permission-denied', 'Permission refusée')

  await ref.update({ status: 'inactive', updatedAt: FieldValue.serverTimestamp() })
  return { ok: true }
})
```

---

## Web Hooks (add to `useProductListings.ts`)

```typescript
export function useUpdateListing() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (params: { listingId: string; updates: Partial<ProductListing> }) =>
      httpsCallable(functions, 'updateProductListing')(params),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['my-listings'] }),
  })
}

export function useDeactivateListing() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (listingId: string) =>
      httpsCallable(functions, 'deactivateProductListing')({ listingId }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['my-listings'] }),
  })
}
```

---

## UI Changes in `AgricultorMarket.tsx`

### "Mes annonces" section
Replace mock `myListings` with `useMyListings()` from `useProductListings.ts` (already built in S8).

Listing card now has functional action buttons:
```
🌽 Maïs                                        ACTIF
   2 500 FC/kg · 3 000 kg disponibles · Qualité A
   Disponible jusqu'au 31 août 2026
   [Modifier]  [Désactiver]
```

**"Modifier" → opens `ModifierAnnonceModal`**:
```
Modifier mon annonce

Prix (FC/kg)         [2 500        ]
Quantité (kg)        [3 000        ]
Disponible jusqu'au  [31 août 2026 ]
Description          [______________]

[Annuler]   [Enregistrer]
```

On save: calls `updateProductListing` CF → invalidates `my-listings` query → closes modal.

**"Désactiver" → confirmation dialog** → calls `deactivateProductListing` CF → listing removed from list.

### bourseTicker strip
Replace mock ticker with `useProductListings()` data: compute ticker from active listings grouped by commodity — show average price per commodity.

---

## Acceptance Criteria
- [ ] `updateProductListing` CF validates ownership + updates allowed fields only
- [ ] `deactivateProductListing` CF marks listing inactive
- [ ] `AgricultorMarket` "Mes annonces" uses `useMyListings()` (real data, not mock)
- [ ] "Modifier" button opens `ModifierAnnonceModal` (not toast stub)
- [ ] `ModifierAnnonceModal` calls `updateProductListing` CF on save
- [ ] "Désactiver" button calls `deactivateProductListing` with confirmation
- [ ] Deactivated listing disappears from the list immediately
- [ ] bourseTicker strip computed from active `product_listings` (not mock)
- [ ] Dev mode: mock listings used, no CF calls

---

## Implementation Status (updated 2026-07-23)

**PARTIAL — real data wired; edit/deactivate CFs and modal not built**

### ✅ Done (this session)
- `AgricultorMarket.tsx` "Mes annonces" tab: replaced `myListings` mock import with `useMyListings()` hook
- `usePricesByProvince()` used for "Prix marché" tab (real CF data)
- Loading spinners and empty states added

### ❌ Remaining
- `updateProductListing` CF not built (not in `mombongo-functions/src/index.ts`)
- `deactivateProductListing` CF not built
- `useUpdateListing()` and `useDeactivateListing()` hooks not added to `useProductListings.ts`
- `ModifierAnnonceModal` component not built — "Modifier" button still shows toast stub
- "Désactiver" button not implemented
- bourseTicker strip still computed from mock data (not from real `product_listings`)
