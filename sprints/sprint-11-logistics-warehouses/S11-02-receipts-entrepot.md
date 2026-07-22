# S11-02 — Reçus d'Entrepôt (Warehouse Receipts)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S11-02 |
| Sprint | Sprint 11 — Logistique & Entrepôts |
| Branch | `feature/s11-02-receipts-entrepot` |
| Merges into | `dev` |
| Estimate | 4h |
| Dependencies | S11-01, S10-00 (exploitation data model) |

---

## Context

A farmer can deposit produce at a certified warehouse (partner) and receive a digital receipt proving their stock. This receipt can be shown to buyers as proof of available goods and potentially used as collateral for financing applications (S4-xx link).

---

## Firestore Collections

### `warehouses` (seeded by admin)
```
{
  id: string
  name: string                      // "Entrepôt Matadi Central"
  province: string
  territory: string
  address: string
  capacityKg: number
  currentUsedKg: number
  commoditiesAccepted: string[]
  ratePerKgPerDayCdf: number        // storage cost
  managerId?: string                // uid of warehouse manager (future role)
  phone: string
  isActive: boolean
}
```

### `warehouse_receipts`
```
{
  id: string
  farmerId: string
  farmerName: string
  warehouseId: string
  warehouseName: string
  commodity: string
  quantityKg: number
  quality: 'A' | 'B' | 'C'
  depositedAt: Timestamp
  expiresAt: Timestamp              // max 90 days
  storageCostPerDayCdf: number
  receiptNumber: string             // human-readable: WR-2026-00042
  status: 'active' | 'redeemed' | 'expired'
  usedAsCollateral?: boolean        // true if linked to a financing application
  collateralApplicationId?: string
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

---

## Cloud Functions

### `getWarehouses`
```typescript
// Params: { province?: string; commodity?: string }
export const getWarehouses = functions.region('europe-west1').https.onCall(async (data) => {
  const snap = await db.collection('warehouses').where('isActive', '==', true).get()
  let warehouses = snap.docs.map(d => ({ id: d.id, ...d.data() })) as any[]
  const { province, commodity } = data ?? {}
  if (province) warehouses = warehouses.filter((w: any) => w.province === province)
  if (commodity) warehouses = warehouses.filter((w: any) =>
    w.commoditiesAccepted.includes(commodity) || w.commoditiesAccepted.includes('tous')
  )
  return { warehouses }
})
```

### `createWarehouseReceipt` (admin-initiated via mombongo-admin)
```typescript
// Called by admin when a physical deposit is confirmed at a partner warehouse
// Params: { farmerId, warehouseId, commodity, quantityKg, quality, daysStorage }
export const createWarehouseReceipt = functions.region('europe-west1').https.onCall(async (data, context) => {
  // Admin-only: verify context.auth?.token.admin === true
  const { farmerId, warehouseId, commodity, quantityKg, quality, daysStorage = 30 } = data

  const warehouseSnap = await db.collection('warehouses').doc(warehouseId).get()
  if (!warehouseSnap.exists) throw new functions.https.HttpsError('not-found', 'Entrepôt introuvable')
  const w = warehouseSnap.data()!

  const farmerSnap = await db.collection('users').doc(farmerId).get()
  const farmerName = farmerSnap.data()?.fullName ?? 'Agriculteur'

  // Generate receipt number
  const countSnap = await db.collection('warehouse_receipts').count().get()
  const receiptNumber = `WR-${new Date().getFullYear()}-${String(countSnap.data().count + 1).padStart(5, '0')}`

  const now = new Date()
  const expiresAt = new Date(now.getTime() + daysStorage * 86400000)

  const ref = db.collection('warehouse_receipts').doc()
  await ref.set({
    farmerId, farmerName,
    warehouseId, warehouseName: w.name,
    commodity, quantityKg, quality,
    depositedAt: Timestamp.fromDate(now),
    expiresAt: Timestamp.fromDate(expiresAt),
    storageCostPerDayCdf: w.ratePerKgPerDayCdf * quantityKg,
    receiptNumber, status: 'active',
    usedAsCollateral: false,
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  })

  // Update warehouse capacity
  await warehouseSnap.ref.update({
    currentUsedKg: FieldValue.increment(quantityKg)
  })

  return { receiptId: ref.id, receiptNumber }
})
```

### `getMyWarehouseReceipts`
```typescript
export const getMyWarehouseReceipts = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')
  const snap = await db.collection('warehouse_receipts')
    .where('farmerId', '==', uid)
    .where('status', '==', 'active')
    .orderBy('depositedAt', 'desc').get()
  return { receipts: snap.docs.map(d => ({ id: d.id, ...d.data() })) }
})
```

---

## Web UI

### "Mes reçus d'entrepôt" (inside Mon Exploitation dashboard, new tab)

```
Mes reçus d'entrepôt

  ┌─────────────────────────────────────────────────┐
  │  WR-2026-00042                          ACTIF   │
  │  🌽 Maïs · 3 000 kg · Qualité A                │
  │  Entrepôt Matadi Central                        │
  │  Déposé le 15 juil. · Expire le 14 août        │
  │  Coût stockage : 1 500 FC/jour                 │
  └─────────────────────────────────────────────────┘

Entrepôts disponibles près de vous
  (cards: name · province · commodities · rate)

[Demander un reçu d'entrepôt]  → info screen explaining the process
```

### Receipt detail / QR code
Each receipt card has a "Voir le reçu" button that shows:
- All receipt fields
- A QR code (base64 encoded receipt ID) that the buyer or admin can scan
- "Utiliser comme garantie" CTA (links to financing application flow, S4-xx)

QR code generation: use `qrcode` npm package, rendered as `<img src="data:image/png;base64,..." />`.

---

## Acceptance Criteria
- [ ] `warehouses` collection seeded with 3 partner warehouses in different provinces
- [ ] `getWarehouses` CF filters by province + commodity accepted
- [ ] `createWarehouseReceipt` is admin-only, generates `receiptNumber` like WR-2026-00042
- [ ] `getMyWarehouseReceipts` returns active receipts for the caller
- [ ] Receipt card in web shows all fields + expiry countdown
- [ ] QR code displays receipt ID as scannable image
- [ ] "Utiliser comme garantie" CTA visible (links to financing, can be a toast stub if S4 not done)
- [ ] `usedAsCollateral: true` shown as a badge on the receipt card
