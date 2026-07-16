# S8-00 — Agro Exchange — Data Model & Seed

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S8-00 |
| Sprint | Sprint 8 — Agro Exchange |
| Branch | `feature/s8-00-agro-exchange-data-model` |
| Merges into | `dev` |
| Estimate | 3h |
| Dependencies | S3-00 (functions project initialized, admin pattern established) |

## Context

The current Bourse implementation covers a **logistics-investment** model (investors fund transport/storage/transformation routes). The new requirement is a full **commodity exchange** (bourse de produits agricoles): farmers list produce → buyers post needs → automatic matching → escrow payment → delivery confirmation. This sprint establishes the data model and seeds Firestore so all later sprints (S8-01 through S8-03) can build on it.

The logistics-investment model (S3-xx) is kept as-is — it becomes the "Opportunités d'investissement" tab inside the Bourse. The commodity exchange becomes the primary "Marché" tab.

---

## Firestore Collections (new)

### `product_listings`
A seller (farmer, cooperative, exporter) posts an offer:
```
{
  id: string
  sellerId: string              // uid
  sellerName: string
  sellerRole: 'farmer' | 'cooperative' | 'exporter' | 'merchant'
  commodity: string             // 'Maïs', 'Manioc', 'Cacao', 'Riz', 'Haricot', ...
  quantityKg: number
  quality: 'A' | 'B' | 'C'    // A = premium, B = standard, C = brut
  province: string              // 'Kinshasa', 'Kongo-Central', 'Équateur', ...
  territory: string             // specific territory within province
  pricePerKgCdf: number         // seller's asking price
  availableFrom: Timestamp
  availableUntil: Timestamp
  photoUrls: string[]           // signed URLs via Storage CF (S8-01)
  certUrl?: string              // quality certificate (optional)
  description?: string
  status: 'active' | 'matched' | 'sold' | 'expired' | 'cancelled'
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

### `buyer_orders`
A buyer (merchant, processor, exporter) posts a demand:
```
{
  id: string
  buyerId: string
  buyerName: string
  buyerRole: 'merchant' | 'processor' | 'exporter' | 'cooperative'
  commodity: string
  quantityKg: number
  maxPricePerKgCdf: number
  deliveryProvince: string
  deliveryTerritory?: string
  neededBy: Timestamp
  description?: string
  status: 'open' | 'matched' | 'contracted' | 'closed'
  createdAt: Timestamp
}
```

### `bourse_matches`
Created by the matching CF or admin when a listing and order are compatible:
```
{
  id: string
  listingId: string
  orderId: string
  sellerId: string
  buyerId: string
  commodity: string
  quantityKg: number            // agreed quantity (min of listing/order qty)
  agreedPricePerKgCdf?: number  // set after negotiation step
  totalCdf?: number
  status: 'pending_negotiation' | 'agreed' | 'contracted' | 'escrowed' | 'shipped' | 'delivered' | 'completed' | 'disputed' | 'cancelled'
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

### `bourse_contracts`
Auto-generated when both parties confirm a match:
```
{
  id: string
  matchId: string
  sellerId: string
  buyerId: string
  commodity: string
  quantityKg: number
  pricePerKgCdf: number
  totalCdf: number
  deliveryLocation: string
  paymentTerms: 'escrow'        // always escrow for now
  deliveryDeadline: Timestamp
  sellerSignedAt?: Timestamp
  buyerSignedAt?: Timestamp
  status: 'pending_signatures' | 'active' | 'fulfilled' | 'disputed'
  createdAt: Timestamp
}
```

### `bourse_prices_by_province`
Daily commodity price per province (updated by admin or computed CF):
```
{
  id: string
  commodity: string
  province: string
  priceCdfPerKg: number
  previousPriceCdfPerKg?: number
  volumeKgTraded: number
  recordedDate: string          // 'YYYY-MM-DD'
  recordedAt: Timestamp
}
```

### `escrow_accounts`
One escrow record per contract:
```
{
  id: string
  contractId: string
  matchId: string
  buyerId: string
  sellerId: string
  amountCdf: number
  depositedAt?: Timestamp
  releasedAt?: Timestamp
  status: 'pending' | 'funded' | 'released' | 'refunded' | 'disputed'
}
```

---

## mombongo-functions

### Step 1 — Seed script

Create `src/scripts/seedAgroExchange.ts`:

```typescript
import { db } from '../lib/admin'
import { FieldValue, Timestamp } from 'firebase-admin/firestore'

const listings = [
  {
    sellerId: 'seed-farmer-1',
    sellerName: 'Jean Mukeba',
    sellerRole: 'farmer',
    commodity: 'Maïs',
    quantityKg: 20_000,
    quality: 'B',
    province: 'Bandundu',
    territory: 'Kikwit',
    pricePerKgCdf: 400,
    availableFrom: Timestamp.fromDate(new Date('2026-07-20')),
    availableUntil: Timestamp.fromDate(new Date('2026-09-01')),
    photoUrls: [],
    status: 'active',
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  },
  {
    sellerId: 'seed-farmer-2',
    sellerName: 'Coopérative Kivu Vert',
    sellerRole: 'cooperative',
    commodity: 'Haricot',
    quantityKg: 5_000,
    quality: 'A',
    province: 'Sud-Kivu',
    territory: 'Uvira',
    pricePerKgCdf: 1_200,
    availableFrom: Timestamp.fromDate(new Date('2026-08-01')),
    availableUntil: Timestamp.fromDate(new Date('2026-10-01')),
    photoUrls: [],
    status: 'active',
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  },
  {
    sellerId: 'seed-farmer-3',
    sellerName: 'Ambroise Kalenga',
    sellerRole: 'farmer',
    commodity: 'Cacao',
    quantityKg: 3_000,
    quality: 'A',
    province: 'Équateur',
    territory: 'Mbandaka',
    pricePerKgCdf: 4_800,
    availableFrom: Timestamp.fromDate(new Date('2026-07-25')),
    availableUntil: Timestamp.fromDate(new Date('2026-08-31')),
    photoUrls: [],
    status: 'active',
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  },
]

const buyerOrders = [
  {
    buyerId: 'seed-merchant-1',
    buyerName: 'Minoterie Fraîcheur SARL',
    buyerRole: 'processor',
    commodity: 'Maïs',
    quantityKg: 15_000,
    maxPricePerKgCdf: 420,
    deliveryProvince: 'Kinshasa',
    neededBy: Timestamp.fromDate(new Date('2026-08-15')),
    status: 'open',
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    buyerId: 'seed-merchant-2',
    buyerName: 'Café Export Congo',
    buyerRole: 'exporter',
    commodity: 'Cacao',
    quantityKg: 2_000,
    maxPricePerKgCdf: 5_000,
    deliveryProvince: 'Kinshasa',
    neededBy: Timestamp.fromDate(new Date('2026-09-01')),
    status: 'open',
    createdAt: FieldValue.serverTimestamp(),
  },
]

const prices = [
  { commodity: 'Maïs',   province: 'Kinshasa',      priceCdfPerKg: 430, volumeKgTraded: 45_000 },
  { commodity: 'Maïs',   province: 'Bandundu',       priceCdfPerKg: 395, volumeKgTraded: 28_000 },
  { commodity: 'Manioc', province: 'Kinshasa',       priceCdfPerKg: 620, volumeKgTraded: 80_000 },
  { commodity: 'Manioc', province: 'Kongo-Central',  priceCdfPerKg: 580, volumeKgTraded: 32_000 },
  { commodity: 'Cacao',  province: 'Équateur',       priceCdfPerKg: 4_800, volumeKgTraded: 8_000 },
  { commodity: 'Haricot', province: 'Sud-Kivu',      priceCdfPerKg: 1_180, volumeKgTraded: 12_000 },
  { commodity: 'Riz',    province: 'Maniema',        priceCdfPerKg: 1_050, volumeKgTraded: 15_000 },
]

async function seed() {
  const batch = db.batch()
  const today = new Date().toISOString().split('T')[0]
  for (const l of listings) batch.set(db.collection('product_listings').doc(), l)
  for (const o of buyerOrders) batch.set(db.collection('buyer_orders').doc(), o)
  for (const p of prices) {
    batch.set(db.collection('bourse_prices_by_province').doc(), {
      ...p,
      recordedDate: today,
      recordedAt: FieldValue.serverTimestamp(),
    })
  }
  await batch.commit()
  console.log('Agro Exchange seeded')
}
seed().catch(console.error)
```

### Step 2 — Read Cloud Functions

Create `src/bourse/getProductListings.ts`:
```typescript
// Returns active product_listings, filtered by commodity/province if provided
// Params: { commodity?: string; province?: string; limit?: number }
// Returns: { listings: ProductListing[] }
export const getProductListings = functions.region('europe-west1').https.onCall(async (data) => {
  const { commodity, province, limit: lim = 20 } = data ?? {}
  let q = db.collection('product_listings').where('status', '==', 'active') as any
  if (commodity) q = q.where('commodity', '==', commodity)
  if (province)  q = q.where('province', '==', province)
  const snap = await q.orderBy('createdAt', 'desc').limit(lim).get()
  return { listings: snap.docs.map((d: any) => ({ id: d.id, ...d.data() })) }
})
```

Create `src/bourse/getBuyerOrders.ts`:
```typescript
// Returns open buyer_orders, filtered by commodity if provided
// Params: { commodity?: string }
// Returns: { orders: BuyerOrder[] }
export const getBuyerOrders = functions.region('europe-west1').https.onCall(async (data) => {
  const { commodity } = data ?? {}
  let q = db.collection('buyer_orders').where('status', '==', 'open') as any
  if (commodity) q = q.where('commodity', '==', commodity)
  const snap = await q.orderBy('createdAt', 'desc').limit(20).get()
  return { orders: snap.docs.map((d: any) => ({ id: d.id, ...d.data() })) }
})
```

Create `src/bourse/getBoursePricesByProvince.ts`:
```typescript
// Returns latest price per commodity per province
// Params: { commodity?: string }
// Returns: { prices: BoursePriceByProvince[] }
export const getBoursePricesByProvince = functions.region('europe-west1').https.onCall(async (data) => {
  const { commodity } = data ?? {}
  let q = db.collection('bourse_prices_by_province')
    .orderBy('recordedAt', 'desc').limit(200) as any
  if (commodity) q = db.collection('bourse_prices_by_province')
    .where('commodity', '==', commodity)
    .orderBy('recordedAt', 'desc').limit(50)
  const snap = await q.get()
  // Return latest price per commodity+province pair
  const latest = new Map<string, any>()
  snap.docs.forEach((d: any) => {
    const key = `${d.data().commodity}|${d.data().province}`
    if (!latest.has(key)) latest.set(key, { id: d.id, ...d.data() })
  })
  return { prices: [...latest.values()] }
})
```

Export all three in `src/index.ts`.

---

## mombongo-admin

### AdminAgroExchange screen

`src/pages/AdminAgroExchange.tsx` — three tabs:

**Tab 1 — Offres (product_listings)**: table: seller / commodity / qty / province / quality / status / price. Actions: view detail, toggle status.

**Tab 2 — Demandes (buyer_orders)**: table: buyer / commodity / qty / maxPrice / province / status. Actions: view detail, close.

**Tab 3 — Prix par province**: table: commodity / province / price / volume / date. "Mettre à jour le prix" button creates a `bourse_prices_by_province` doc.

---

## ✅ Definition of Done
- [ ] Seed script populates `product_listings`, `buyer_orders`, `bourse_prices_by_province`
- [ ] `getProductListings`, `getBuyerOrders`, `getBoursePricesByProvince` CFs deployed
- [ ] Admin `/agro-exchange` shows all three tabs with live data
- [ ] `npm run build` exits 0 (functions + admin)

```bash
firebase deploy --only functions:getProductListings,functions:getBuyerOrders,functions:getBoursePricesByProvince
git commit -m "feat(s8-00): agro exchange data model — collections, seed + read CFs"
```
