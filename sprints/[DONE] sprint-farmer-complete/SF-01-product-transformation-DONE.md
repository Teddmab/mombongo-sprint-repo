# SF-01 — Post-Harvest Product Transformation

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SF-01 |
| Sprint | Sprint Farmer Complete — Product Transformation |
| Branch | `feature/sf-01-product-transformation` |
| Merges into | `main` |
| Estimate | 5h |
| Dependencies | S10-00 (exploitation + culture data model), S8-01 (createProductListing CF) |

---

## Why this matters

DRC farmers almost never sell raw produce at maximum value. The biggest jump in value happens at transformation:
- **Manioc → Farine de manioc**: 1 kg manioc → 0.25 kg farine, but farine sells 4–6× the price per kg
- **Maïs → Farine de maïs**: 1 kg maïs → 0.7 kg farine
- **Palmier à huile → Huile de palme**: 10 kg régimes → 1 L huile
- **Cacao → Fèves fermentées/séchées**: sorting + fermentation doubles export price
- **Café → Café lavé**: wet processing increases cup quality grade

Currently there is **no way** in Mombongo to record a transformation, track its costs, compute yield, or publish the transformed product. A farmer who transforms must exit the app to manage this.

This sprint adds a full transformation record-and-publish flow accessible two ways:
1. **From a CultureCard**: a "Transformer" button appears once harvest month is reached
2. **From a standalone `/transformation` route**: visible in the farmer nav at all times (for transforming purchased raw materials too)

---

## Firestore Collection

### `product_transformations`

```typescript
{
  id: string

  // Identity
  farmerId: string
  exploitationId?: string     // if sourced from own exploitation
  cultureId?: string          // if sourced from a specific culture record

  // Raw input
  rawCommodity: string        // "Manioc"
  rawQuantityKg: number       // 1 000 kg
  rawQuality: 'A' | 'B' | 'C'

  // Transformation
  transformationType:
    | 'mouture'         // grinding → flour
    | 'fermentation'    // cacao, café, attiéké
    | 'séchage'         // sun-drying
    | 'pressage'        // palm oil extraction
    | 'fumage'          // smoking fish/meat
    | 'autre'

  transformedProduct: string  // "Farine de manioc"
  transformedQuantityKg: number  // 250 kg
  yieldPct: number            // computed: (250/1000) * 100 = 25%

  // Costs (all in CDF)
  processingCostCdf: number   // moulin / processor fee
  laborCostCdf: number        // own or hired labor
  transportCostCdf: number    // raw material to moulin + product back
  packagingCostCdf: number    // bags, bottles, labels
  totalCostCdf: number        // sum — computed before saving
  costPerKgCdf: number        // totalCostCdf / transformedQuantityKg — for pricing reference

  // Output details
  packagingUnit: 'sac 50kg' | 'sac 25kg' | 'bidon 20L' | 'kg vrac' | 'autre'
  unitCount: number           // e.g., 5 sacs de 50kg
  transformedAt: string       // ISO date of transformation

  // Processor info (optional)
  processorName?: string      // "Moulin Kikwit Centre"
  processorLocation?: string  // "Kikwit, Kwilu"

  // Lifecycle
  status: 'recorded' | 'listed' | 'sold'
  listingId?: string          // set when createListingFromTransformation is called

  notes?: string
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

---

## Cloud Functions

### `recordProductTransformation`

```typescript
// mombongo-functions/src/farmer/recordProductTransformation.ts
export const recordProductTransformation = functions.region('europe-west1').https.onCall(
  async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const {
      rawCommodity, rawQuantityKg, rawQuality,
      transformationType, transformedProduct, transformedQuantityKg,
      processingCostCdf = 0, laborCostCdf = 0,
      transportCostCdf = 0, packagingCostCdf = 0,
      packagingUnit, unitCount, transformedAt,
      processorName, processorLocation,
      exploitationId, cultureId, notes,
    } = data

    if (rawQuantityKg <= 0) throw new functions.https.HttpsError('invalid-argument', 'Quantité source invalide')
    if (transformedQuantityKg <= 0) throw new functions.https.HttpsError('invalid-argument', 'Quantité produit invalide')
    if (transformedQuantityKg > rawQuantityKg) throw new functions.https.HttpsError('invalid-argument', 'Le produit transformé ne peut dépasser la source')

    const totalCostCdf = processingCostCdf + laborCostCdf + transportCostCdf + packagingCostCdf
    const yieldPct = Math.round((transformedQuantityKg / rawQuantityKg) * 100)
    const costPerKgCdf = transformedQuantityKg > 0 ? Math.round(totalCostCdf / transformedQuantityKg) : 0

    const ref = db.collection('product_transformations').doc()
    await ref.set({
      farmerId: uid,
      exploitationId: exploitationId ?? null,
      cultureId: cultureId ?? null,
      rawCommodity, rawQuantityKg, rawQuality,
      transformationType, transformedProduct, transformedQuantityKg,
      yieldPct, processingCostCdf, laborCostCdf, transportCostCdf,
      packagingCostCdf, totalCostCdf, costPerKgCdf,
      packagingUnit, unitCount: unitCount ?? 1,
      transformedAt,
      processorName: processorName ?? null,
      processorLocation: processorLocation ?? null,
      status: 'recorded',
      listingId: null,
      notes: notes ?? '',
      createdAt: FieldValue.serverTimestamp(),
      updatedAt: FieldValue.serverTimestamp(),
    })

    return { transformationId: ref.id, yieldPct, totalCostCdf, costPerKgCdf }
  }
)
```

### `getMyProductTransformations`

```typescript
export const getMyProductTransformations = functions.region('europe-west1').https.onCall(
  async (_, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const snap = await db.collection('product_transformations')
      .where('farmerId', '==', uid)
      .orderBy('createdAt', 'desc')
      .limit(50)
      .get()

    return { transformations: snap.docs.map(d => ({ id: d.id, ...d.data() })) }
  }
)
```

### `createListingFromTransformation`

Reuses the existing `product_listings` collection from S8-01. Updates the transformation doc to link the listing.

```typescript
export const createListingFromTransformation = functions.region('europe-west1').https.onCall(
  async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const { transformationId, pricePerKgCdf, province, territory, availableUntil, description } = data

    const txSnap = await db.collection('product_transformations').doc(transformationId).get()
    if (!txSnap.exists) throw new functions.https.HttpsError('not-found', 'Transformation introuvable')
    const tx = txSnap.data()!
    if (tx.farmerId !== uid) throw new functions.https.HttpsError('permission-denied', 'Non autorisé')
    if (tx.status !== 'recorded') throw new functions.https.HttpsError('failed-precondition', 'Déjà publié')

    const userSnap = await db.collection('users').doc(uid).get()
    const sellerName = userSnap.data()?.displayName ?? 'Agriculteur'

    const listingRef = db.collection('product_listings').doc()
    const now = FieldValue.serverTimestamp()

    const batch = db.batch()

    // Create the listing (reusing product_listings collection from S8-01)
    batch.set(listingRef, {
      sellerId: uid,
      sellerName,
      sellerRole: 'farmer',
      commodity: tx.transformedProduct,
      quantityKg: tx.transformedQuantityKg,
      quality: tx.rawQuality,
      province,
      territory,
      pricePerKgCdf,
      availableFrom: new Date().toISOString(),
      availableUntil: new Date(availableUntil).toISOString(),
      description: description ?? `${tx.rawQuantityKg} kg de ${tx.rawCommodity} transformés. Rendement: ${tx.yieldPct}%.`,
      photoUrls: [],
      status: 'active',
      sourceTransformationId: transformationId,   // link back
      createdAt: now,
      updatedAt: now,
    })

    // Mark transformation as listed
    batch.update(txSnap.ref, {
      status: 'listed',
      listingId: listingRef.id,
      updatedAt: now,
    })

    await batch.commit()
    return { listingId: listingRef.id }
  }
)
```

Export all 3 in `src/index.ts`.

---

## mombongo-web

### Hook — `src/hooks/useProductTransformations.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions, isDevMode } from '@/lib/firebase'

export interface ProductTransformation {
  id: string
  farmerId: string
  exploitationId?: string
  cultureId?: string
  rawCommodity: string
  rawQuantityKg: number
  rawQuality: 'A' | 'B' | 'C'
  transformationType: string
  transformedProduct: string
  transformedQuantityKg: number
  yieldPct: number
  processingCostCdf: number
  laborCostCdf: number
  transportCostCdf: number
  packagingCostCdf: number
  totalCostCdf: number
  costPerKgCdf: number
  packagingUnit: string
  unitCount: number
  transformedAt: string
  processorName?: string
  processorLocation?: string
  status: 'recorded' | 'listed' | 'sold'
  listingId?: string
  notes?: string
  createdAt: { seconds: number }
}

const MOCK_TRANSFORMATIONS: ProductTransformation[] = [
  {
    id: 'tx-1', farmerId: 'mock-uid',
    rawCommodity: 'Manioc', rawQuantityKg: 1000, rawQuality: 'B',
    transformationType: 'mouture', transformedProduct: 'Farine de manioc',
    transformedQuantityKg: 250, yieldPct: 25,
    processingCostCdf: 15000, laborCostCdf: 5000, transportCostCdf: 3000, packagingCostCdf: 2000,
    totalCostCdf: 25000, costPerKgCdf: 100,
    packagingUnit: 'sac 50kg', unitCount: 5,
    transformedAt: new Date(Date.now() - 7 * 86400000).toISOString(),
    processorName: 'Moulin Kikwit Centre', processorLocation: 'Kikwit, Kwilu',
    status: 'recorded',
    notes: 'Bon rendement cette saison',
    createdAt: { seconds: Math.floor((Date.now() - 7 * 86400000) / 1000) },
  },
]

export function useMyProductTransformations() {
  return useQuery({
    queryKey: ['my-transformations'],
    queryFn: async (): Promise<ProductTransformation[]> => {
      if (isDevMode()) return MOCK_TRANSFORMATIONS
      const call = httpsCallable<Record<string, never>, { transformations: ProductTransformation[] }>(
        functions, 'getMyProductTransformations'
      )
      return (await call({})).data.transformations
    },
    staleTime: 2 * 60_000,
  })
}

export function useRecordTransformation() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: async (payload: Omit<ProductTransformation, 'id' | 'farmerId' | 'yieldPct' | 'totalCostCdf' | 'costPerKgCdf' | 'status' | 'listingId' | 'createdAt'>): Promise<{ transformationId: string; yieldPct: number; totalCostCdf: number; costPerKgCdf: number }> => {
      if (isDevMode()) return { transformationId: 'tx-' + Date.now(), yieldPct: 25, totalCostCdf: 25000, costPerKgCdf: 100 }
      const call = httpsCallable<typeof payload, { transformationId: string; yieldPct: number; totalCostCdf: number; costPerKgCdf: number }>(
        functions, 'recordProductTransformation'
      )
      return (await call(payload)).data
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ['my-transformations'] }),
  })
}

export function useCreateListingFromTransformation() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: async (payload: { transformationId: string; pricePerKgCdf: number; province: string; territory: string; availableUntil: string; description?: string }): Promise<{ listingId: string }> => {
      if (isDevMode()) return { listingId: 'listing-' + Date.now() }
      const call = httpsCallable<typeof payload, { listingId: string }>(
        functions, 'createListingFromTransformation'
      )
      return (await call(payload)).data
    },
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['my-transformations'] })
      qc.invalidateQueries({ queryKey: ['my-listings'] })
    },
  })
}
```

### Route — `/transformation`

New page `src/pages/TransformationScreen.tsx` (farmer-only, dispatched from the top-level router):

```
Entry from CultureCard: when moisRecolte <= currentMonth, show "Transformer" button
  → navigates to /transformation?cultureId=xxx&commodity=Manioc&qty=1000

Entry from nav: /transformation (direct)

TransformationScreen layout:
├── Header: "Mes transformations" + "+ Nouvelle transformation" button
├── Transformation list (cards):
│   ├── TransformationCard: rawCommodity → transformedProduct, yieldPct, date, status chip
│   │   └── If status = 'recorded': "📢 Publier" button → PublierTransformationModal
│   └── Empty state: "Aucune transformation — enregistrez votre première"
└── NouvelleTransformationModal (3-step):
    ├── Step 1: Source
    │   ├── From my culture: exploitation + culture dropdown (pre-filled from URL params if any)
    │   ├── Or: manual entry (commodity text + quantity + quality)
    │   └── Transformation type: grid of 6 buttons (mouture / fermentation / séchage / pressage / fumage / autre)
    ├── Step 2: Produit obtenu
    │   ├── Transformed product name (e.g., "Farine de manioc" — pre-suggested based on type)
    │   ├── Quantity obtained (kg)
    │   ├── Yield display: "Rendement : 25%" (auto-computed, shown in color)
    │   ├── Packaging: unit + count
    │   ├── Date of transformation (date picker)
    │   └── Processor: name + location (optional)
    └── Step 3: Coûts
        ├── Processing fee (FC)
        ├── Labor (FC)
        ├── Transport (FC)
        ├── Packaging (FC)
        ├── TOTAL auto-computed + Cost/kg display
        └── Notes (optional)

PublierTransformationModal (after recording OR from "Publier" button):
  ├── Pre-filled: product = transformedProduct, quantity = transformedQuantityKg
  ├── Price per kg (FC) — shown alongside costPerKgCdf for margin reference
  ├── Province + Territory
  ├── Available until (date)
  └── Submit → createListingFromTransformation CF → toast + card shows "PUBLIÉ" chip
```

### CultureCard update

In `src/pages/exploitation/cultures/CultureCard.tsx`, when `culture.status === 'active'` and the current month >= `culture.moisRecolte`, show:

```tsx
<Link to={`/transformation?cultureId=${culture.id}&commodity=${culture.commodity}&qty=${culture.productionAttenduKg}`}
  className="h-8 px-3 bg-amber-50 text-amber-700 rounded-lg font-bold text-[11px] flex items-center gap-1.5">
  🔄 Transformer
</Link>
```

### Router update

In `src/App.tsx` (or wherever routes are defined), add:
```tsx
{ path: '/transformation', element: <TransformationScreen /> }
```

Render only for farmer role (role dispatch at screen level).

---

## DRC-Specific Transformation Presets

To help farmers fill the form quickly, provide pre-filled suggestions based on `rawCommodity + transformationType`:

| Input | Type | Suggested Output |
|-------|------|-----------------|
| Manioc | Mouture | Farine de manioc |
| Maïs | Mouture | Farine de maïs |
| Palmier à huile | Pressage | Huile de palme |
| Cacao | Fermentation | Fèves de cacao fermentées |
| Café | Fermentation | Café lavé |
| Manioc | Fermentation | Chikwangue / Foufou |
| Soja | Mouture | Farine de soja |

Store as `TRANSFORMATION_PRESETS` in a constants file, used to pre-fill "transformedProduct" when the user picks a combination.

---

## Acceptance Criteria

- [ ] `recordProductTransformation` CF creates `product_transformations/{id}` with computed yieldPct, totalCostCdf, costPerKgCdf
- [ ] `getMyProductTransformations` CF returns transformations for the authenticated farmer
- [ ] `createListingFromTransformation` CF creates `product_listings/{id}` linked to transformation + updates transformation.status → 'listed'
- [ ] `/transformation` route accessible from farmer nav
- [ ] CultureCard shows "Transformer" button when current month >= moisRecolte
- [ ] CultureCard "Transformer" pre-fills commodity + qty in the form
- [ ] 3-step modal: Step 3 shows auto-computed yield%, total cost, cost/kg
- [ ] Transformation presets: selecting rawCommodity + transformationType auto-fills transformedProduct
- [ ] After recording: "📢 Publier sur le marché" CTA visible
- [ ] After publishing: transformation card shows "PUBLIÉ" chip, listingId set
- [ ] Dev mode: mock transformations shown without CF calls
- [ ] `npx tsc --noEmit` passes with 0 errors
- [ ] `npx vitest run` passes
