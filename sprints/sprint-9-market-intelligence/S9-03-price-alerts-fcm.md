# S9-03 — Price Alerts via FCM

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S9-03 |
| Sprint | Sprint 9 — Market Intelligence |
| Branch | `feature/s9-03-price-alerts` |
| Merges into | `dev` |
| Estimate | 3h |
| Dependencies | S9-01 (price history), FCM tokens registered (S6-02) |

## Context

Farmers need to know the right moment to sell. This story lets them set a target price: "Notify me when Maïs in Kinshasa reaches 450 FC/kg." A scheduled CF checks all alerts daily and sends FCM pushes for any that trigger.

---

## Firestore Collection: `price_alerts`

```
{
  id: string
  userId: string
  commodity: string
  province: string
  targetPriceCdf: number
  direction: 'above' | 'below'   // notify when price goes above OR below target
  status: 'active' | 'triggered' | 'cancelled'
  lastCheckedAt?: Timestamp
  triggeredAt?: Timestamp
  createdAt: Timestamp
}
```

---

## Cloud Functions

### `createPriceAlert` (new)
```typescript
// Params: { commodity, province, targetPriceCdf, direction }
export const createPriceAlert = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { commodity, province, targetPriceCdf, direction = 'above' } = data
  if (!(targetPriceCdf > 0)) throw new functions.https.HttpsError('invalid-argument', 'Prix invalide')

  // max 5 active alerts per user
  const existing = await db.collection('price_alerts')
    .where('userId', '==', uid).where('status', '==', 'active').get()
  if (existing.size >= 5) throw new functions.https.HttpsError('resource-exhausted', 'Maximum 5 alertes actives')

  const ref = db.collection('price_alerts').doc()
  await ref.set({
    userId: uid, commodity, province, targetPriceCdf, direction,
    status: 'active',
    createdAt: FieldValue.serverTimestamp(),
  })
  return { alertId: ref.id }
})
```

### `cancelPriceAlert` (new)
```typescript
export const cancelPriceAlert = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')
  const alertSnap = await db.collection('price_alerts').doc(data.alertId).get()
  if (alertSnap.data()?.userId !== uid) throw new functions.https.HttpsError('permission-denied', 'Non autorisé')
  await alertSnap.ref.update({ status: 'cancelled' })
  return { ok: true }
})
```

### `getMyPriceAlerts` (new)
```typescript
export const getMyPriceAlerts = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')
  const snap = await db.collection('price_alerts')
    .where('userId', '==', uid)
    .orderBy('createdAt', 'desc').limit(10).get()
  return { alerts: snap.docs.map(d => ({ id: d.id, ...d.data() })) }
})
```

### `checkPriceAlerts` (scheduled — daily)
```typescript
export const checkPriceAlerts = functions
  .region('europe-west1')
  .pubsub.schedule('0 8 * * *')
  .timeZone('Africa/Kinshasa')
  .onRun(async () => {
    const alertsSnap = await db.collection('price_alerts').where('status', '==', 'active').get()
    const now = FieldValue.serverTimestamp()

    for (const alertDoc of alertsSnap.docs) {
      const alert = alertDoc.data()
      // Get latest price for this commodity+province
      const priceSnap = await db.collection('bourse_prices_by_province')
        .where('commodity', '==', alert.commodity)
        .where('province', '==', alert.province)
        .orderBy('recordedAt', 'desc').limit(1).get()

      if (priceSnap.empty) continue
      const currentPrice = priceSnap.docs[0].data().priceCdfPerKg

      const triggered = alert.direction === 'above'
        ? currentPrice >= alert.targetPriceCdf
        : currentPrice <= alert.targetPriceCdf

      if (!triggered) {
        await alertDoc.ref.update({ lastCheckedAt: now })
        continue
      }

      // Trigger: update status + send FCM
      await alertDoc.ref.update({ status: 'triggered', triggeredAt: now })

      const userSnap = await db.collection('users').doc(alert.userId).get()
      const tokens: string[] = userSnap.data()?.fcmTokens ?? []
      if (tokens.length === 0) continue

      const { getMessaging } = await import('firebase-admin/messaging')
      await getMessaging().sendMulticast({
        tokens,
        notification: {
          title: `Alerte prix — ${alert.commodity}`,
          body: `${alert.commodity} à ${currentPrice.toLocaleString()} FC/kg à ${alert.province} — votre objectif de ${alert.targetPriceCdf.toLocaleString()} FC/kg est atteint !`,
        },
        data: { type: 'price_alert', commodity: alert.commodity, province: alert.province },
      })
    }
  })
```

---

## Web UI

### Alert management (in BourseScreen "Prix" tab or Profile)

**"Mes alertes prix"** — expandable section:
- List of active alerts: commodity · province · target price · direction chip
- "Annuler" button per alert
- "+ Créer une alerte" button → opens `PrixAlerteModal`

### `PrixAlerteModal`
Simple 3-field form:
```
Produit:       [Maïs ▾]
Province:      [Kinshasa ▾]
M'alerter si le prix dépasse:  [___] FC/kg
              [Au-dessus ▾ / En-dessous ▾]

[Créer l'alerte]
```

Max 5 alerts displayed with disabled state + tooltip "Maximum 5 alertes actives" if exceeded.

---

## Acceptance Criteria
- [ ] `createPriceAlert` CF validates max 5 per user, creates doc
- [ ] `cancelPriceAlert` CF marks alert as cancelled
- [ ] `checkPriceAlerts` scheduled CF runs daily, triggers FCM for matching alerts
- [ ] Alert list visible in web app with create/cancel UI
- [ ] `PrixAlerteModal` form validates all fields before calling CF
- [ ] Dev mode: create/cancel operations show toast without real CF call
