# S9-01 — Province Price Map & Daily Price Index

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S9-01 |
| Sprint | Sprint 9 — Market Intelligence |
| Branch | `feature/s9-01-province-price-map` |
| Merges into | `dev` |
| Estimate | 5h |
| Dependencies | S8-00 (`bourse_prices_by_province` collection exists) |

## Context

Farmers and merchants currently have no visibility into commodity prices across DRC provinces. They rely on word of mouth or middlemen. This story surfaces live price data per province + a 30-day trend chart, giving every actor on the platform a credible price reference.

---

## Data Model

Uses existing `bourse_prices_by_province` collection (created in S8-00):
```
{
  commodity: string        // 'Maïs', 'Manioc', 'Cacao', ...
  province: string
  priceCdfPerKg: number
  previousPriceCdfPerKg?: number
  volumeKgTraded: number
  recordedDate: string     // 'YYYY-MM-DD'
  recordedAt: Timestamp
}
```

### New collection: `price_history`
To power the 30-day chart, a scheduled CF writes daily snapshots:
```
{
  commodity: string
  province: string
  priceCdfPerKg: number
  volumeKgTraded: number
  date: string             // 'YYYY-MM-DD'
  recordedAt: Timestamp
}
```

---

## Cloud Functions (mombongo-functions)

### `getBoursePricesByProvince` (already exists — no change)
Returns latest price per commodity+province pair. Used by the price table.

### `getPriceHistory` (new)
```typescript
// Params: { commodity: string; province?: string; days?: number }
// Returns: { history: { date: string; priceCdfPerKg: number; volumeKgTraded: number }[] }
export const getPriceHistory = functions.region('europe-west1').https.onCall(async (data) => {
  const { commodity, province, days = 30 } = data ?? {}
  const since = new Date()
  since.setDate(since.getDate() - days)

  let q = db.collection('price_history')
    .where('commodity', '==', commodity)
    .where('recordedAt', '>=', since)
    .orderBy('recordedAt', 'asc')
    .limit(days + 5) as any
  if (province) q = q.where('province', '==', province)

  const snap = await q.get()
  return { history: snap.docs.map((d: any) => d.data()) }
})
```

### `scheduleDailyPriceSnapshot` (new — Pub/Sub scheduled CF)
```typescript
// Runs daily at 23:30 — copies current bourse_prices_by_province to price_history
export const scheduleDailyPriceSnapshot = functions
  .region('europe-west1')
  .pubsub.schedule('30 23 * * *')
  .timeZone('Africa/Kinshasa')
  .onRun(async () => {
    const snap = await db.collection('bourse_prices_by_province').get()
    const batch = db.batch()
    const today = new Date().toISOString().split('T')[0]
    snap.docs.forEach(d => {
      const ref = db.collection('price_history').doc()
      batch.set(ref, { ...d.data(), date: today, recordedAt: FieldValue.serverTimestamp() })
    })
    await batch.commit()
  })
```

---

## Web UI (mombongo-web)

### New screen: `PrixMarcheScreen` (or tab within BourseScreen)

**Route**: `/prix-marche` (or BourseScreen "Prix" tab)

**Desktop layout:**
```
┌─────────────────────────────────────────────────────────┐
│  Prix par province     [Maïs ▾]  [Toutes provinces ▾]   │
├──────────────────┬──────────────────────────────────────┤
│ TABLE                          │  CHART                  │
│ Province │ Prix │ Var │ Volume │  30-day trend (recharts)│
│ Kinshasa │ 430  │ +2% │ 45T    │  AreaChart avec tooltip │
│ Bandundu │ 395  │ -1% │ 28T    │  Cliquer une ligne →   │
│ Sud-Kivu │1 180 │ +5% │ 12T    │  change le graphique   │
└──────────────────┴──────────────────────────────────────┘
```

**Mobile layout:**
- Dropdown to select commodity
- Scrollable card list: province + prix + variation badge
- Tap a province → modal/expand with 30-day mini-chart

### Web hook: `usePriceHistory`
```typescript
// src/hooks/useBoursePrices.ts (extend existing file)
export function usePriceHistory(commodity: string, province?: string) {
  return useQuery({
    queryKey: ['price-history', commodity, province],
    queryFn: async () => {
      if (isDevMode()) return MOCK_PRICE_HISTORY
      const call = httpsCallable(functions, 'getPriceHistory')
      return (await call({ commodity, province, days: 30 })).data
    },
    staleTime: 5 * 60_000,
  })
}
```

### Mock data (dev mode)
```typescript
// src/data/mock.ts — add:
export const MOCK_PRICE_HISTORY = {
  history: Array.from({ length: 30 }, (_, i) => ({
    date: new Date(Date.now() - (29 - i) * 86400000).toISOString().split('T')[0],
    priceCdfPerKg: 400 + Math.round(Math.sin(i / 5) * 30 + Math.random() * 15),
    volumeKgTraded: 20000 + Math.round(Math.random() * 15000),
  }))
}
```

### Price variation badge component
```tsx
function PriceBadge({ current, previous }: { current: number; previous?: number }) {
  if (!previous) return null
  const pct = ((current - previous) / previous) * 100
  const up = pct >= 0
  return (
    <span className={`text-[11px] font-bold flex items-center gap-0.5 ${up ? 'text-green-600' : 'text-red-500'}`}>
      {up ? '▲' : '▼'} {Math.abs(pct).toFixed(1)}%
    </span>
  )
}
```

---

## Admin (mombongo-admin)

### AdminAgroExchange — "Prix" tab (already exists, expand)
- Add "Mettre à jour le prix" form:
  - Province dropdown + commodity dropdown + price input + volume input
  - Calls `updateDoc` on the matching `bourse_prices_by_province` doc (creates if missing)
- Show last updated timestamp per row

---

## i18n Keys (add to fr.json + ln.json)

```json
"prix": {
  "title": "Prix du marché",
  "byProvince": "Prix par province",
  "selectCommodity": "Choisir un produit",
  "allProvinces": "Toutes les provinces",
  "trend30d": "Tendance 30 jours",
  "volumeTraded": "Volume échangé",
  "lastUpdated": "Mis à jour",
  "noData": "Pas de données disponibles"
}
```

---

## Acceptance Criteria
- [ ] `getPriceHistory` CF returns 30-day price array for a commodity+province
- [ ] `scheduleDailyPriceSnapshot` creates a new `price_history` doc each day
- [ ] BourseScreen (or `/prix-marche`) shows price table filterable by commodity
- [ ] Clicking a row shows 30-day recharts AreaChart with tooltips
- [ ] Mobile: tap a province card → mini-chart appears
- [ ] Price variation badge shows % change vs previous day (green/red)
- [ ] `npx tsc --noEmit` passes in both `mombongo-web` and `mombongo-functions`
- [ ] Dev mode shows mock data without Firebase calls
