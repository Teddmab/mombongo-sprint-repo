# SF-04 — Personalized Crop Price Alerts (Per-Farmer Thresholds)

## Goal
Farmer subscribes to price alerts for their specific crops and province with a threshold and direction (above/below). More targeted than the generic province-level S9-03 alerts. A scheduled CF scans province price updates daily and fires FCM when a threshold is crossed.

## Status
**TODO** · Priority P1 · Est. 3h · Web

## Branch
`feature/sf-04-crop-price-alerts`

## Repos
- `mombongo-web`
- `mombongo-functions`

---

## Firestore

### Collection: `farmer_price_alerts`
```
farmer_price_alerts/{alertId}
  farmerId: string               // uid
  commodity: string              // e.g. "Manioc"
  province: string               // e.g. "Kinshasa"
  thresholdUsd: number           // e.g. 0.35
  direction: 'above' | 'below'
  active: boolean
  lastTriggeredAt?: Timestamp
  createdAt: Timestamp
```

**Uniqueness constraint:** one alert per `(farmerId, commodity, direction)` — enforced in CF using `set` with merge on a deterministic doc ID: `${uid}_${commodity}_${direction}`.

---

## Cloud Functions

### `setFarmerPriceAlert` (callable)
**Input:**
```typescript
{
  commodity: string
  province: string
  thresholdUsd: number           // must be > 0
  direction: 'above' | 'below'
}
```
**Logic:** `docId = ${uid}_${commodity}_${direction}`; `set` with merge on `farmer_price_alerts/{docId}`. Sets `active: true`, updates `thresholdUsd`, `province`.

**Returns:** `{ alertId: docId }`

---

### `deleteFarmerPriceAlert` (callable)
**Input:** `{ alertId: string }`
**Logic:** verify `farmerId === uid`, hard delete doc.

---

### `getMyPriceAlerts` (callable)
**Input:** _(none)_
**Output:** Array of `farmer_price_alerts` where `farmerId === uid && active === true`, ordered by `createdAt` desc.

---

### `checkFarmerPriceAlerts` (scheduled — daily, 08:00 DRC time)
**Logic:**
1. Fetch all `province_prices` updated in the last 24h
2. For each price update, query `farmer_price_alerts` where `commodity === price.commodity && province === price.province && active === true`
3. For each matching alert:
   - `direction === 'above'` and `price.priceUsd >= alert.thresholdUsd` → trigger
   - `direction === 'below'` and `price.priceUsd <= alert.thresholdUsd` → trigger
4. Send FCM via existing `notificationService.sendFcm()`: title = "📈 Alerte prix Manioc", body = "Prix actuel: $0.38/kg — votre seuil de $0.35 atteint"
5. Update `lastTriggeredAt` on alert doc

---

## Frontend

### Hook: `src/hooks/useFarmerPriceAlerts.ts`
```typescript
export function useMyPriceAlerts(): FarmerPriceAlert[]
export function useSetFarmerPriceAlert(): UseMutationResult<...>
export function useDeleteFarmerPriceAlert(): UseMutationResult<...>
```

Mock guard:
```typescript
if (isDevMode()) return [MOCK_PRICE_ALERTS]
```

`MOCK_PRICE_ALERTS`:
```typescript
[
  { alertId: 'pa-001', commodity: 'Manioc', province: 'Kinshasa', thresholdUsd: 0.35, direction: 'above', active: true },
  { alertId: 'pa-002', commodity: 'Maïs',   province: 'Kinshasa', thresholdUsd: 0.20, direction: 'below', active: true },
]
```

---

### UI: `AgricultorBourse.tsx`

Add **"Mes alertes prix"** collapsible section below the price table:

**Alert chip list:**
- Each chip shows: `[Manioc] Au-dessus de $0.35 — Kinshasa` + ✕ delete button (2-click confirm)
- Empty state: "Aucune alerte configurée" + CTA

**"+ Ajouter une alerte" button** → `AddPriceAlertModal`

#### `AddPriceAlertModal`
- **Commodity** — `<select>` populated from `useMyCultures()` + manual entry option
- **Province** — `<select>` of 26 DRC provinces (default: `user.province`)
- **Direction** — toggle: "Au-dessus de / En-dessous de"
- **Seuil (USD/kg)** — number input, step 0.01, min 0.01
- Live preview: "Vous serez alerté quand le prix de Manioc dépasse $0.35/kg à Kinshasa"
- Submit → `setFarmerPriceAlert` mutation

---

## Smoke Tests
1. Bourse tab: "Mes alertes prix" collapsible section renders with 2 mock alerts
2. Add alert modal: fill + submit → `farmer_price_alerts/{alertId}` created in Firestore
3. Delete alert: confirm → chip disappears, `active = false` (or doc deleted)
4. `checkFarmerPriceAlerts` emulator test: simulate `province_prices` update crossing threshold → FCM notification body verified
5. `npx tsc --noEmit` passes in `mombongo-web`
