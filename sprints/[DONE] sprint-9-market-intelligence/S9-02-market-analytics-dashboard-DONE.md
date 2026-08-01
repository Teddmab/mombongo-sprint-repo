# S9-02 — Market Analytics Dashboard

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S9-02 |
| Sprint | Sprint 9 — Market Intelligence |
| Branch | `feature/s9-02-market-analytics` |
| Merges into | `dev` |
| Estimate | 3h |
| Dependencies | S9-01 (price history data exists), S8-03 (bourse_contracts) |

## Context

All roles (farmer, merchant, investor, agent) benefit from understanding market activity: which commodities are most traded, when demand peaks, which province has the highest volume. This story creates a "Tendances du marché" tab or screen powered by a single aggregation CF.

---

## Cloud Function

### `getMarketStats` (new)
```typescript
// Params: { period?: 'week' | 'month' }
// Returns: { stats: MarketStats }
export const getMarketStats = functions.region('europe-west1').https.onCall(async (data) => {
  const period = data?.period ?? 'month'
  const since = new Date()
  since.setDate(since.getDate() - (period === 'week' ? 7 : 30))

  const [contractsSnap, ordersSnap, listingsSnap] = await Promise.all([
    db.collection('bourse_contracts')
      .where('status', '==', 'fulfilled')
      .where('createdAt', '>=', since).get(),
    db.collection('buyer_orders')
      .where('createdAt', '>=', since).get(),
    db.collection('product_listings')
      .where('createdAt', '>=', since).get(),
  ])

  // Volume per commodity
  const volumeByCommodity: Record<string, { kg: number; valueCdf: number; count: number }> = {}
  contractsSnap.docs.forEach(d => {
    const { commodity, quantityKg, totalCdf } = d.data()
    if (!volumeByCommodity[commodity]) volumeByCommodity[commodity] = { kg: 0, valueCdf: 0, count: 0 }
    volumeByCommodity[commodity].kg += quantityKg
    volumeByCommodity[commodity].valueCdf += totalCdf
    volumeByCommodity[commodity].count++
  })

  // Demand by month (neededBy month from buyer_orders)
  const demandByMonth: Record<string, number> = {}
  ordersSnap.docs.forEach(d => {
    const neededBy = d.data().neededBy?.toDate?.()
    if (neededBy) {
      const month = neededBy.toISOString().slice(0, 7) // 'YYYY-MM'
      demandByMonth[month] = (demandByMonth[month] ?? 0) + 1
    }
  })

  const totalContractsCdf = contractsSnap.docs.reduce((sum, d) => sum + (d.data().totalCdf ?? 0), 0)

  return {
    stats: {
      period,
      totalFulfilledContracts: contractsSnap.size,
      totalContractValueCdf: totalContractsCdf,
      totalActiveListings: listingsSnap.size,
      totalOpenOrders: ordersSnap.size,
      volumeByCommodity,
      demandByMonth,
    }
  }
})
```

---

## Web UI

### New tab inside BourseScreen: "Tendances"

**Top KPI strip** (4 cards):
- Contrats conclus ce mois
- Valeur totale échangée (FC)
- Annonces actives
- Demandes ouvertes

**Volume by commodity** — horizontal bar chart (recharts `BarChart`):
- Sorted by kg traded descending
- Each bar shows kg + value in tooltip
- Color-coded by commodity category

**Demand calendar** — simple monthly grid:
- Each month = a cell showing number of buyer orders needing delivery
- Highlighted cells = peak demand months
- Helps farmers plan planting dates

**Top 3 rising commodities** — small cards with trend arrow + price delta

### Mock data (dev mode)
```typescript
export const MOCK_MARKET_STATS = {
  stats: {
    period: 'month',
    totalFulfilledContracts: 14,
    totalContractValueCdf: 48_600_000,
    totalActiveListings: 27,
    totalOpenOrders: 19,
    volumeByCommodity: {
      'Maïs':   { kg: 32_000, valueCdf: 12_800_000, count: 5 },
      'Manioc': { kg: 18_000, valueCdf: 11_160_000, count: 4 },
      'Cacao':  { kg: 4_000,  valueCdf: 19_200_000, count: 2 },
      'Haricot':{ kg: 6_000,  valueCdf: 7_200_000,  count: 3 },
    },
    demandByMonth: {
      '2026-08': 8, '2026-09': 12, '2026-10': 6,
      '2026-11': 9, '2026-12': 14, '2027-01': 5,
    },
  }
}
```

---

## Acceptance Criteria
- [ ] `getMarketStats` CF returns aggregated data for week/month
- [ ] "Tendances" tab visible in BourseScreen (investor + merchant + farmer views)
- [ ] 4 KPI cards at top with real/mock data
- [ ] BarChart shows volume by commodity
- [ ] Demand calendar highlights peak months
- [ ] Dev mode uses mock data, no Firebase calls
