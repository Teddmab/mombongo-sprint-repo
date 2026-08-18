# SFA-01-03 — Market Prices Screen (Real Data)

## Context
The current mobile market screen (`MarketScreen.tsx`) already has role-based dispatch and uses `useProducts` which calls `getProducts` CF. However, for a farmer, the "market" view is primarily about **current commodity prices** (province_prices) rather than product listings. The farmer needs a price-first view: "What is maize worth today in my province?"

This sprint creates a farmer-specific market view that surfaces price data from `province_prices` via CF.

## Scope
- Create `src/hooks/useMarketPrices.ts` — calls `getMarketPrices` CF (returns province_prices for farmer's province + nearby)
- Create `src/screens/market/FarmerMarketScreen.tsx` — price-first layout
- Show the farmer's primary crop price prominently at top
- List other commodity prices below
- Show delta indicators (↑↓) when `previousPricePerKgCdf` is available

## Cloud Function required
`getMarketPrices` — input: `{ province?, crops?: string[] }` → output:
```typescript
{
  prices: Array<{
    commodity: string
    province: string
    pricePerKgCdf: number
    previousPricePerKgCdf?: number
    deltaPercent?: number
    updatedAt: string
  }>
}
```
This CF queries the `province_prices` collection. Gate with `isDevMode()` mock.

## Files to create / modify
- `src/hooks/useMarketPrices.ts`
- `src/screens/market/FarmerMarketScreen.tsx`
- `src/screens/MarketScreen.tsx` — route to `FarmerMarketScreen` when `isFarmerApp`

## Implementation

### `src/hooks/useMarketPrices.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

const MOCK_PRICES = [
  { commodity: 'Maïs', province: 'Kinshasa', pricePerKgCdf: 450, deltaPercent: 5 },
  { commodity: 'Manioc', province: 'Kinshasa', pricePerKgCdf: 320, deltaPercent: -2 },
  { commodity: 'Haricots', province: 'Kinshasa', pricePerKgCdf: 820, deltaPercent: 0 },
]

export function useMarketPrices(province?: string) {
  return useQuery({
    queryKey: ['marketPrices', province],
    queryFn: async () => {
      if (isDevMode()) return MOCK_PRICES
      const res = await httpsCallable<{ province?: string }, { prices: typeof MOCK_PRICES }>(
        functions, 'getMarketPrices'
      )({ province })
      return res.data.prices
    },
    staleTime: 30 * 60_000,
  })
}
```

### FarmerMarketScreen.tsx — key layout
```typescript
// Primary crop card at top (larger, with delta indicator)
// Scrollable list of other commodity prices below
// "Dernière mise à jour: Aujourd'hui 06h30" footer
```

## Acceptance criteria
- [ ] Farmer's primary crop price appears prominently at top
- [ ] ↑/↓ delta shown when `deltaPercent` is non-zero
- [ ] Other commodity prices listed below
- [ ] Pull-to-refresh triggers re-fetch
- [ ] Dev mode shows mock data

## Smoke test
1. Open Market tab as farmer — confirm price-first layout (not product listings)
2. Verify delta arrow shows for commodities with a price change
3. Pull to refresh — no crash, data refreshes
