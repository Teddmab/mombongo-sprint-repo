# SF-03 — Farm Revenue & Profitability Analytics Dashboard

## Goal
Season-by-season P&L per crop: revenue from bourse/market sales minus input costs (SF-05) + transformation costs (SF-01) + transport. Crop-by-crop profitability ranking. Answers the core question every DRC farmer has: "Est-ce que ma culture m'a rapporté quelque chose cette saison?"

## Status
**TODO** · Priority P2 · Est. 5h · Web

## Branch
`feature/sf-03-farm-analytics`

## Repos
- `mombongo-web`
- `mombongo-functions`

---

## Cloud Functions

### `getFarmPnlSummary`
**Callable** · farmerId inferred from auth

**Input:**
```typescript
{
  season?: '2024A' | '2024B' | '2025A' | '2025B'  // omit = current season
}
```

**Logic (aggregate across collections):**
- **Revenue** (`revenueUsd`): sum of `bourse_contracts.sellerAmountUsd` where `sellerId === uid` and `status === 'completed'` within season date range
- **Input costs** (`inputCostCdf`): sum of `farm_inputs.costCdf` where `farmerId === uid` within season date range
- **Transformation costs** (`transformCostCdf`): sum of `product_transformations.totalCostCdf` where `farmerId === uid` within season date range
- Compute `totalCostCdf`, convert to USD at cached exchange rate, compute `netProfitUsd`, `marginPct`

**Output:**
```typescript
{
  season: string
  revenueUsd: number
  totalCostCdf: number
  totalCostUsd: number          // converted at cached rate
  netProfitUsd: number
  marginPct: number
  exchangeRate: number
}
```

---

### `getCropProfitability`
**Callable** · farmerId inferred from auth

**Input:**
```typescript
{
  season?: string
}
```

**Output:** Array ranked by `marginPct` desc:
```typescript
[{
  commodity: string
  revenueUsd: number
  totalCostCdf: number
  netProfitUsd: number
  marginPct: number
  harvestCount: number          // from harvest_records
}]
```

---

## Frontend

### Hook: `src/hooks/useFarmAnalytics.ts`
```typescript
export function useFarmPnlSummary(season?: string): FarmPnlSummary
export function useCropProfitability(season?: string): CropProfitabilityRow[]
```

Mock data:
```typescript
const MOCK_FARM_PNL = {
  season: '2025A',
  revenueUsd: 420,
  totalCostCdf: 210000,
  totalCostUsd: 75,
  netProfitUsd: 345,
  marginPct: 82,
  exchangeRate: 2800,
}

const MOCK_CROP_PROFITABILITY = [
  { commodity: 'Manioc', revenueUsd: 280, totalCostCdf: 140000, netProfitUsd: 230, marginPct: 82, harvestCount: 2 },
  { commodity: 'Maïs',   revenueUsd: 140, totalCostCdf: 70000,  netProfitUsd: 115, marginPct: 82, harvestCount: 1 },
]
```

---

### Page: `src/pages/FarmAnalyticsScreen.tsx`
**Route:** `/farm-analytics` · Farmer-only guard

#### Entry point
Mon Exploitation screen: add **"📊 Mes analyses"** card (same visual weight as other quick-action cards) → navigates to `/farm-analytics`

#### Layout

**Header:** Season selector chips (2024B / 2025A / 2025B), default = current

**Summary stat cards (4):**
| Card | Value | Color |
|---|---|---|
| Revenus | `$420` | green |
| Coûts totaux | `75,000 CDF` | red |
| Bénéfice net | `$345` | green |
| Marge | `82%` | amber |

**Crop ranking table:**
- Columns: Commodity, Revenue, Costs, Net Profit, Margin %, # Harvests
- Rows sorted by margin desc
- Row tap → expands cost breakdown inline

**Cost breakdown donut chart** (use `<canvas>` + simple arc drawing, no external lib):
- Segments: Inputs / Transformation / Transport / Packaging
- Labels below with CDF amounts

---

## Smoke Tests
1. Mon Exploitation: "📊 Mes analyses" card visible → tapping loads `/farm-analytics`
2. 4 summary cards render with non-zero mock values
3. Crop ranking table: ≥ 2 rows in dev mode
4. Season selector: switching season triggers CF call (or mock swap) and updates cards
5. `npx tsc --noEmit` passes in `mombongo-web`
