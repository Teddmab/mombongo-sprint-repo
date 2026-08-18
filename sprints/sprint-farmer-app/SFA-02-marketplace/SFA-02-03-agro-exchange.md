# SFA-02-03 — AgroExchange Screen (Farmer App)

## Context
Port of SM-06 (AgroExchange) adapted for the Farmer App. The AgroExchange is the commodity order book (buy/sell orders for commodities). For a farmer, the primary use case is:
1. View current bids for their commodity
2. Place a sell order (price + quantity + delivery window)
3. See their open orders and status

The web Sprint 8 (S8) implemented AgroExchange CFs: `getOrderBook`, `placeOrder`, `getMyOrders`, `cancelOrder`, `getContractStatus`.

## Scope
- Create `src/screens/exchange/AgroExchangeScreen.tsx`
- Create `src/screens/exchange/PlaceOrderSheet.tsx`
- Create `src/hooks/useOrderBook.ts` — calls `getOrderBook` CF
- Create `src/hooks/useMyOrders.ts` — calls `getMyOrders` CF
- Add "Échange" tab to Farmer App (or accessible from Bourse as sub-tab)

## Cloud Functions required (all already deployed from S8)
- `getOrderBook` — returns active buy/sell orders for a commodity
- `placeOrder` — submit a sell order
- `getMyOrders` — farmer's open orders
- `cancelOrder` — cancel an open order

## Files to create
- `src/hooks/useOrderBook.ts`
- `src/hooks/useMyOrders.ts`
- `src/screens/exchange/AgroExchangeScreen.tsx`
- `src/screens/exchange/PlaceOrderSheet.tsx`

## Implementation

### `src/hooks/useOrderBook.ts`
```typescript
export function useOrderBook(commodity: string) {
  return useQuery({
    queryKey: ['orderBook', commodity],
    queryFn: async () => {
      if (isDevMode()) return MOCK_ORDER_BOOK
      const res = await httpsCallable<{ commodity: string }, { bids: Order[]; asks: Order[] }>(
        functions, 'getOrderBook'
      )({ commodity })
      return res.data
    },
    staleTime: 5 * 60_000,
    refetchInterval: 5 * 60_000, // refresh every 5 min
  })
}
```

### AgroExchangeScreen.tsx
```typescript
// Tab strip: Maïs | Manioc | Haricots | Riz
// Order book table: bids (buy orders from merchants/investors) vs asks
// "Je veux vendre" FAB → PlaceOrderSheet
// My orders section below
```

### PlaceOrderSheet.tsx
```typescript
// Quantity (kg), Price per kg (CDF), Delivery window (weeks)
// Submit → placeOrder CF
// On success: invalidate myOrders + orderBook
```

## Acceptance criteria
- [ ] AgroExchange tab/screen accessible from Farmer App
- [ ] Order book loads bids and asks for selected commodity
- [ ] Farmer can place a sell order via the sheet
- [ ] My open orders listed below the order book
- [ ] Cancel button on open orders calls `cancelOrder` CF
- [ ] Auto-refreshes every 5 min

## Smoke test
1. Open AgroExchange — select "Maïs" tab
2. Confirm order book shows bids (dev: mock data; live: real orders)
3. Tap "Je veux vendre" → fill price + quantity → submit
4. Confirm order appears in "My orders" section
5. Cancel the order — confirm it disappears from the list
