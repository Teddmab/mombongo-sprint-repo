# S8-02 — Agro Exchange — Buyer Order Book & Match Discovery

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S8-02 |
| Sprint | Sprint 8 — Agro Exchange |
| Branch | `feature/s8-02-order-book-matching` |
| Merges into | `dev` |
| Estimate | 5h |
| Dependencies | S8-01 (listings exist in Firestore) |

## Context

After sellers list their produce, buyers (merchants, processors, exporters) must be able to:
1. Browse seller listings and post their own demand orders
2. See potential matches between their demand and available supply
3. Initiate a negotiation / direct contact with a seller

This sprint also adds the "Marketplace" tab to `BourseScreen` (investor view) so all roles can browse available commodities, and creates the `PublierDemandeModal` for merchant/buyer roles.

**Price discovery model for MVP**: Direct negotiation only (not auction). The seller sets an asking price; the buyer proposes a counter-price; they agree on a final price. Full auction mechanism is S11-01 (later).

---

## mombongo-functions

### createBuyerOrder onCall

Create `src/bourse/createBuyerOrder.ts`:

```typescript
export const createBuyerOrder = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const {
    commodity, quantityKg, maxPricePerKgCdf, deliveryProvince, deliveryTerritory, neededBy, description,
  } = data as {
    commodity: string; quantityKg: number; maxPricePerKgCdf: number
    deliveryProvince: string; deliveryTerritory?: string; neededBy: string; description?: string
  }

  const userSnap = await db.collection('users').doc(uid).get()
  const buyerName = userSnap.data()?.displayName ?? 'Acheteur'
  const buyerRole = userSnap.data()?.role ?? 'merchant'

  const ref = db.collection('buyer_orders').doc()
  const now = admin.firestore.FieldValue.serverTimestamp()

  await ref.set({
    buyerId: uid, buyerName, buyerRole,
    commodity, quantityKg, maxPricePerKgCdf,
    deliveryProvince, deliveryTerritory: deliveryTerritory ?? '',
    neededBy: new Date(neededBy),
    description: description ?? '',
    status: 'open',
    createdAt: now,
  })

  // Auto-match: find active listings with the same commodity and price <= maxPricePerKgCdf
  const matchSnap = await db.collection('product_listings')
    .where('status', '==', 'active')
    .where('commodity', '==', commodity)
    .get()

  const candidates = matchSnap.docs.filter(d => d.data().pricePerKgCdf <= maxPricePerKgCdf)

  if (candidates.length > 0) {
    // Create bourse_matches for the top 3 candidates
    const batch = db.batch()
    const topMatches = candidates.slice(0, 3)
    for (const c of topMatches) {
      const matchRef = db.collection('bourse_matches').doc()
      batch.set(matchRef, {
        listingId: c.id,
        orderId: ref.id,
        sellerId: c.data().sellerId,
        buyerId: uid,
        commodity,
        quantityKg: Math.min(quantityKg, c.data().quantityKg),
        status: 'pending_negotiation',
        createdAt: now,
        updatedAt: now,
      })
    }
    await batch.commit()
  }

  return { orderId: ref.id, matchCount: candidates.length }
})
```

### getMatches onCall

Create `src/bourse/getMatches.ts`:

```typescript
// Returns bourse_matches where the calling user is buyer or seller
// Params: { role: 'buyer' | 'seller' }
// Returns: { matches: BourseMatch[] }
export const getMatches = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const role: 'buyer' | 'seller' = data?.role ?? 'buyer'
  const field = role === 'buyer' ? 'buyerId' : 'sellerId'

  const snap = await db.collection('bourse_matches')
    .where(field, '==', uid)
    .where('status', 'in', ['pending_negotiation', 'agreed'])
    .orderBy('createdAt', 'desc')
    .limit(20)
    .get()

  return { matches: snap.docs.map(d => ({ id: d.id, ...d.data() })) }
})
```

### proposePrice onCall

Create `src/bourse/proposePrice.ts`:

```typescript
// Either party proposes a price to the other.
// Creates a sub-collection `bourse_matches/{id}/negotiations` doc.
// Params: { matchId: string; proposedPricePerKgCdf: number; message?: string }
export const proposePrice = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { matchId, proposedPricePerKgCdf, message } = data as {
    matchId: string; proposedPricePerKgCdf: number; message?: string
  }

  const matchRef = db.collection('bourse_matches').doc(matchId)
  const matchSnap = await matchRef.get()
  if (!matchSnap.exists) throw new functions.https.HttpsError('not-found', 'Match introuvable')

  const match = matchSnap.data()!
  if (match.sellerId !== uid && match.buyerId !== uid)
    throw new functions.https.HttpsError('permission-denied', 'Non autorisé')
  if (match.status !== 'pending_negotiation')
    throw new functions.https.HttpsError('failed-precondition', 'Négociation fermée')

  const proposedBy: 'seller' | 'buyer' = match.sellerId === uid ? 'seller' : 'buyer'
  const now = admin.firestore.FieldValue.serverTimestamp()

  await matchRef.collection('negotiations').doc().set({
    proposedBy,
    proposedByUid: uid,
    proposedPricePerKgCdf,
    message: message ?? '',
    createdAt: now,
  })

  return { success: true }
})
```

### acceptPrice onCall

Create `src/bourse/acceptPrice.ts`:

```typescript
// The receiving party accepts the last proposed price.
// Updates bourse_matches.status = 'agreed', sets agreedPricePerKgCdf and totalCdf.
// Params: { matchId: string }
export const acceptPrice = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { matchId } = data as { matchId: string }

  await db.runTransaction(async tx => {
    const matchRef = db.collection('bourse_matches').doc(matchId)
    const matchSnap = await tx.get(matchRef)
    if (!matchSnap.exists) throw new functions.https.HttpsError('not-found', 'Match introuvable')

    const match = matchSnap.data()!
    if (match.sellerId !== uid && match.buyerId !== uid)
      throw new functions.https.HttpsError('permission-denied', 'Non autorisé')

    // Get last negotiation proposal
    const negsSnap = await matchRef.collection('negotiations').orderBy('createdAt', 'desc').limit(1).get()
    if (negsSnap.empty) throw new functions.https.HttpsError('failed-precondition', 'Aucune proposition')

    const lastProposal = negsSnap.docs[0].data()
    const agreedPricePerKgCdf: number = lastProposal.proposedPricePerKgCdf
    const totalCdf = agreedPricePerKgCdf * match.quantityKg
    const now = admin.firestore.FieldValue.serverTimestamp()

    tx.update(matchRef, {
      status: 'agreed',
      agreedPricePerKgCdf,
      totalCdf,
      updatedAt: now,
    })
  })

  return { success: true }
})
```

Export all in `src/index.ts`.

---

## mombongo-web

### Step 1 — useBuyerOrders + useMatches hooks

Create `src/hooks/useBuyerOrders.ts`:

```typescript
export function useBuyerOrders(filters: { commodity?: string } = {}) {
  return useQuery({
    queryKey: ['buyer-orders', filters],
    queryFn: async () => {
      if (isDevMode()) return [] as BuyerOrder[]
      const fn = httpsCallable<any, { orders: BuyerOrder[] }>(functions, 'getBuyerOrders')
      const result = await fn(filters)
      return result.data.orders
    },
    staleTime: 60_000,
  })
}

export function useMyMatches(role: 'buyer' | 'seller') {
  return useQuery({
    queryKey: ['my-matches', role],
    queryFn: async () => {
      if (isDevMode()) return [] as BourseMatch[]
      const fn = httpsCallable<any, { matches: BourseMatch[] }>(functions, 'getMatches')
      const result = await fn({ role })
      return result.data.matches
    },
    staleTime: 30_000,
  })
}
```

### Step 2 — Marketplace tab in BourseScreen (investor view)

Add a "Marché" tab to the investor `DesktopBourse` / `MobileBourse` in `BourseScreen.tsx`:

**Tab layout** — two sub-tabs:
- **Offres** (product_listings): filtered by commodity/province dropdown. Card per listing showing seller, commodity, quantity, quality badge, province, price/kg, availability dates. "Contacter le vendeur" button → opens NegotiationModal (see below).
- **Demandes** (buyer_orders): list of open buyer orders. Each shows buyer role, commodity, quantity, max price, delivery province. "Je peux fournir" button → opens MettreEnVenteModal pre-filled with that commodity.

### Step 3 — PublierDemandeModal (for MerchantBourse + investor)

New modal in `src/components/forms/ActionForms.tsx`:

```typescript
export function PublierDemandeModal({ open, onClose }: { open: boolean; onClose: () => void }) {
  // Same pattern as MettreEnVenteModal but calls createBuyerOrder CF
  // Fields: commodity, quantityKg, maxPricePerKgCdf, deliveryProvince, deliveryTerritory, neededBy, description
  // Shows matchCount in success screen: "X offres correspondent à votre demande"
}
```

Wire it in `MerchantBourse.tsx` replacing the `PublierLotModal` stub.

### Step 4 — NegotiationModal

New modal `src/components/NegotiationModal.tsx`:

```typescript
// Shows the match details, negotiation history (list of proposals), and a form to propose a new price.
// "Accepter ce prix" button calls acceptPrice CF.
// Polls getMatches every 15s to detect when the other party has responded.
interface NegotiationModalProps {
  open: boolean
  onClose: () => void
  matchId: string
  role: 'buyer' | 'seller'
  commodity: string
  quantityKg: number
  sellerPricePerKgCdf: number
  buyerMaxPricePerKgCdf: number
}
```

### Step 5 — i18n keys

```
bourse.marketplace      → "Marché" / "Marketplace"
bourse.listings         → "Offres disponibles" / "Available offers"
bourse.orders           → "Demandes d'achat" / "Purchase orders"
bourse.postDemand       → "Publier une demande" / "Post a demand"
bourse.contactSeller    → "Contacter le vendeur" / "Contact seller"
bourse.iCanSupply       → "Je peux fournir" / "I can supply"
bourse.negotiate        → "Négocier" / "Negotiate"
bourse.proposePrice     → "Proposer un prix" / "Propose a price"
bourse.acceptPrice      → "Accepter ce prix" / "Accept this price"
bourse.matchFound       → "{count} offres correspondent" / "{count} offers match"
```

---

## ✅ Definition of Done
- [ ] `createBuyerOrder` CF saves to `buyer_orders` and auto-creates `bourse_matches`
- [ ] `getMatches` CF returns buyer/seller matches
- [ ] `proposePrice` + `acceptPrice` CFs work end-to-end
- [ ] BourseScreen investor view has "Marché" tab showing live listings + orders
- [ ] `PublierDemandeModal` replaces `PublierLotModal` stub in MerchantBourse
- [ ] `NegotiationModal` displays negotiation history and allows counter-proposal
- [ ] `npm run build` exits 0
- [ ] `npx vitest run` passes

```bash
firebase deploy --only functions:createBuyerOrder,functions:getMatches,functions:proposePrice,functions:acceptPrice
git commit -m "feat(s8-02): buyer order book + match discovery + negotiation CFs"
```
