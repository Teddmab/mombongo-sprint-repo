# S11-01 — Logistique — Partenaires de Transport

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S11-01 |
| Sprint | Sprint 11 — Logistique & Entrepôts |
| Branch | `feature/s11-01-logistique-partenaires` |
| Merges into | `dev` |
| Estimate | 4h |
| Dependencies | S8-03 (bourse_contracts, confirmShipment exists) |

---

## Context

After S8-03, a seller confirms shipment manually. There is no connection to physical transport. This story lets a seller choose a registered transport partner after the escrow is funded, and records the booking on the contract.

---

## Firestore Collections

### `transport_partners` (seeded by admin)
```
{
  id: string
  name: string                      // "Transco SARL"
  phone: string
  coverageProvinces: string[]       // provinces they operate in
  commodities: string[]             // commodities they transport ('Maïs', 'tous', ...)
  ratePerKmCdf: number              // approximate cost per km
  maxCapacityKg: number
  vehicleTypes: string[]            // ['camion 5T', 'pick-up']
  rating: number                    // 1–5, maintained by admin
  isActive: boolean
}
```

### `shipment_bookings` (sub-field on contract, or separate collection)
```
{
  id: string
  contractId: string
  sellerId: string
  buyerId: string
  transportPartnerId: string
  partnerName: string
  partnerPhone: string
  estimatedPickupDate?: string
  estimatedDeliveryDate?: string
  notes?: string
  status: 'booked' | 'picked_up' | 'in_transit' | 'delivered'
  createdAt: Timestamp
}
```

---

## Cloud Functions

### `getTransportPartners`
```typescript
// Params: { province?: string; commodity?: string }
// Returns: { partners: TransportPartner[] }
export const getTransportPartners = functions.region('europe-west1').https.onCall(async (data) => {
  const { province, commodity } = data ?? {}
  const snap = await db.collection('transport_partners')
    .where('isActive', '==', true).get()
  let partners = snap.docs.map(d => ({ id: d.id, ...d.data() })) as any[]
  if (province) partners = partners.filter((p: any) =>
    p.coverageProvinces.includes(province) || p.coverageProvinces.includes('tous')
  )
  if (commodity) partners = partners.filter((p: any) =>
    p.commodities.includes(commodity) || p.commodities.includes('tous')
  )
  return { partners }
})
```

### `bookTransport`
```typescript
// Params: { contractId, transportPartnerId, estimatedPickupDate?, notes? }
export const bookTransport = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { contractId, transportPartnerId, estimatedPickupDate, notes } = data
  const contractSnap = await db.collection('bourse_contracts').doc(contractId).get()
  if (!contractSnap.exists) throw new functions.https.HttpsError('not-found', 'Contrat introuvable')

  const c = contractSnap.data()!
  if (c.sellerId !== uid) throw new functions.https.HttpsError('permission-denied', 'Vendeur uniquement')
  if (c.status !== 'funded') throw new functions.https.HttpsError('failed-precondition', 'Séquestre non financé')

  const partnerSnap = await db.collection('transport_partners').doc(transportPartnerId).get()
  if (!partnerSnap.exists) throw new functions.https.HttpsError('not-found', 'Partenaire introuvable')

  const p = partnerSnap.data()!
  const now = FieldValue.serverTimestamp()
  const bookingRef = db.collection('shipment_bookings').doc()

  const batch = db.batch()
  batch.set(bookingRef, {
    contractId, sellerId: c.sellerId, buyerId: c.buyerId,
    transportPartnerId, partnerName: p.name, partnerPhone: p.phone,
    estimatedPickupDate: estimatedPickupDate ?? null,
    notes: notes ?? '', status: 'booked', createdAt: now,
  })
  batch.update(db.collection('bourse_contracts').doc(contractId), {
    transportPartnerId, transportPartnerName: p.name,
    bookingId: bookingRef.id, updatedAt: now,
  })
  await batch.commit()

  return { bookingId: bookingRef.id }
})
```

---

## Web UI

### "Choisir un transporteur" step (added to ContractModal after escrow funded)

When `contract.status === 'funded'` and seller is viewing their ContractModal, a new section appears:

```
Étape suivante : Organiser le transport
─────────────────────────────────────────────────
Transporteurs disponibles dans votre zone
(filtrés par province de l'exploitation + commodity)

  ┌────────────────────────────────────────────┐
  │ Transco SARL                  ⭐⭐⭐⭐⭐   │
  │ Camion 5T · jusqu'à 8 000 kg              │
  │ ☎ +243 81 234 5678                         │
  │ ~450 FC/km                    [Choisir]   │
  └────────────────────────────────────────────┘
  ┌────────────────────────────────────────────┐
  │ Mbote Transport               ⭐⭐⭐⭐     │
  │ Pick-up · jusqu'à 1 500 kg               │
  │ ☎ +243 99 876 5432                         │
  │ ~320 FC/km                    [Choisir]   │
  └────────────────────────────────────────────┘

[Je gère le transport moi-même → ]  (skips this step, goes to confirmShipment)
```

Tapping "Choisir" → confirmation modal with date picker for estimated pickup → calls `bookTransport` CF → contract shows partner name + phone number for buyer reference.

---

## Acceptance Criteria
- [ ] `transport_partners` collection seeded with 3–5 mock partners via admin or seed script
- [ ] `getTransportPartners` CF filters by province/commodity
- [ ] `bookTransport` CF creates `shipment_bookings` doc and updates contract
- [ ] ContractModal shows transport section when `status === 'funded'` and caller is seller
- [ ] Transport partner cards show name, capacity, rate, rating, phone
- [ ] "Je gère moi-même" skips to `confirmShipment` without booking
- [ ] Dev mode: mock list of 2 partners shown without CF calls
