# S11-03 — Suivi de Livraison

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S11-03 |
| Sprint | Sprint 11 — Logistique & Entrepôts |
| Branch | `feature/s11-03-suivi-livraison` |
| Merges into | `dev` |
| Estimate | 3h |
| Dependencies | S8-03 (confirmShipment), S11-01 (bookTransport) |

---

## Context

After S8-03, the shipment flow is: seller calls `confirmShipment` → buyer calls `confirmDelivery`. There is no intermediate status visibility. Buyers can't see whether the goods have been picked up or are in transit. This story adds intermediate status updates and a visible timeline on the contract detail screen.

---

## Data Changes

Add `shipmentStatus` field to `bourse_contracts`:
```
shipmentStatus?: 'pending' | 'booked' | 'in_transit' | 'arrived'
```
(Separate from `contract.status` which tracks the payment/escrow lifecycle)

---

## Cloud Function

### `updateShipmentStatus`
```typescript
// Params: { contractId, shipmentStatus: 'in_transit' | 'arrived', notes? }
// Only the seller can call this.
export const updateShipmentStatus = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { contractId, shipmentStatus, notes } = data
  const VALID = ['in_transit', 'arrived'] as const
  if (!VALID.includes(shipmentStatus))
    throw new functions.https.HttpsError('invalid-argument', 'Statut invalide')

  const contractRef = db.collection('bourse_contracts').doc(contractId)

  await db.runTransaction(async tx => {
    const snap = await tx.get(contractRef)
    if (!snap.exists) throw new functions.https.HttpsError('not-found', 'Contrat introuvable')
    const c = snap.data()!
    if (c.sellerId !== uid) throw new functions.https.HttpsError('permission-denied', 'Vendeur uniquement')
    if (!['shipped', 'funded'].includes(c.status))
      throw new functions.https.HttpsError('failed-precondition', 'Expédition non confirmée')

    const now = FieldValue.serverTimestamp()
    tx.update(contractRef, {
      shipmentStatus,
      [`shipmentStatusHistory.${shipmentStatus}`]: now,
      notes: notes ?? c.notes ?? '',
      updatedAt: now,
    })
  })

  // Send FCM to buyer
  const contractSnap = await contractRef.get()
  const c = contractSnap.data()!
  const buyerSnap = await db.collection('users').doc(c.buyerId).get()
  const tokens: string[] = buyerSnap.data()?.fcmTokens ?? []
  if (tokens.length > 0) {
    const { getMessaging } = await import('firebase-admin/messaging')
    const statusLabel = shipmentStatus === 'in_transit' ? 'en route' : 'arrivée à destination'
    await getMessaging().sendMulticast({
      tokens,
      notification: {
        title: `Livraison ${statusLabel}`,
        body: `Votre commande de ${c.commodity} est ${statusLabel}.`,
      },
      data: { type: 'shipment_status', contractId, shipmentStatus },
    })
  }

  return { ok: true }
})
```

---

## Web UI

### Contract timeline (added to ContractModal / ContractDetailScreen)

A vertical stepper showing the full lifecycle:

```
  ✅ Contrat signé           12 juil. 2026
  ✅ Séquestre financé       13 juil. 2026
  ✅ Expédié                 15 juil. 2026
  🔄 En transit              16 juil. 2026  ← current
  ○  Arrivé à destination    —
  ○  Livraison confirmée     —
```

Color coding:
- ✅ Green: completed step
- 🔄 Amber pulse: current active step
- ○ Gray: future step

### Seller controls (when `status === 'shipped'`)

Below the timeline, the seller sees:

```
Mettre à jour le statut de livraison

  [📦 Marchandise récupérée / En transit]
  [🏁 Arrivée à destination]

Notes (optionnel): [_______________________]
```

Each button calls `updateShipmentStatus`. Buttons are disabled once the buyer calls `confirmDelivery` (contract `status === 'fulfilled'`).

### Buyer view

Buyer sees the same timeline but with read-only status. When `shipmentStatus === 'arrived'`, they see a prominent:

```
┌──────────────────────────────────────────────┐
│  📦 La marchandise est arrivée à destination │
│  Vérifiez la livraison et confirmez.         │
│                                              │
│  [✓ Confirmer la livraison]                  │
│  (libère le paiement au vendeur)             │
└──────────────────────────────────────────────┘
```

This "Confirmer la livraison" button calls the existing `confirmDelivery` CF (S8-03).

---

## Acceptance Criteria
- [ ] `updateShipmentStatus` CF validates seller auth + valid status values
- [ ] CF sends FCM notification to buyer on each status change
- [ ] ContractModal shows vertical timeline with all lifecycle steps
- [ ] Timeline shows timestamps for completed steps, dashes for pending
- [ ] Seller sees status update buttons when `status === 'shipped'`
- [ ] Buyer sees "Confirmer la livraison" CTA when `shipmentStatus === 'arrived'`
- [ ] Buttons disable after `confirmDelivery` is called (contract fulfilled)
- [ ] Dev mode: timeline shows mock status history
