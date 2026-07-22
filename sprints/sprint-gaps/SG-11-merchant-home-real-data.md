# SG-11 — Merchant Home & Market: Real Data Wiring

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-11 |
| Sprint | Sprint Gaps 11 |
| Branch | `feature/sg-11-merchant-home-real-data` |
| Merges into | `dev` |
| Estimate | 6h |
| Dependencies | SG-07 (wire merchant modals), S8-01 (useProductListings) |

---

## Context

Three merchant screens pull everything from mock:

- **`MerchantHome.tsx`**: `merchantOrders` (order list), `bourseTicker` (price ticker), `products` — all mock
- **`MerchantMarket.tsx`**: `products`, `bourseTicker` — all mock; `CommanderModal` = setTimeout
- **`MerchantBourse.tsx`**: `bourseTicker`, `bourseOpportunities` — all mock; `PublierLotModal`, `ReserverLotModal` = setTimeout; mobile has no `PublierLotModal`

SG-07 defines the CF specs for the modals. This story wires the home page data and the market product listing, and adds the missing mobile `PublierLotModal`.

---

## Cloud Functions

### `getMerchantOrders`
```typescript
// Returns the merchant's own product_orders (from `createProductOrder` — SG-07)
export const getMerchantOrders = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const snap = await db.collection('product_orders')
    .where('merchantId', '==', uid)
    .orderBy('createdAt', 'desc')
    .limit(20)
    .get()

  return { orders: snap.docs.map(d => ({ id: d.id, ...d.data() })) }
})
```

### `getMerchantHomeData`
Single aggregation for the home screen KPIs:
```typescript
// Returns: { orders, kpis, recentListings }
export const getMerchantHomeData = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const [ordersSnap, listingsSnap] = await Promise.all([
    db.collection('product_orders').where('merchantId', '==', uid)
      .orderBy('createdAt', 'desc').limit(5).get(),
    db.collection('product_listings')
      .where('status', '==', 'active')
      .orderBy('createdAt', 'desc').limit(10).get(),
  ])

  const orders = ordersSnap.docs.map(d => ({ id: d.id, ...d.data() }))
  const listings = listingsSnap.docs.map(d => ({ id: d.id, ...d.data() }))

  const totalSpentCdf = orders.reduce((sum: number, o: any) => sum + (o.totalAmountCdf ?? 0), 0)
  const pendingOrders = orders.filter((o: any) => o.status === 'pending').length

  return {
    recentOrders: orders,
    recentListings: listings,
    kpis: {
      totalOrders: ordersSnap.size,
      pendingOrders,
      totalSpentCdf,
    }
  }
})
```

---

## Web Hooks

`src/hooks/useMerchantData.ts`:
```typescript
export function useMerchantHome() {
  if (isDevMode()) return { isLoading: false, recentOrders: MOCK_MERCHANT_ORDERS, recentListings: MOCK_PRODUCTS, kpis: MOCK_MERCHANT_KPIS }
  return useQuery({
    queryKey: ['merchant-home'],
    queryFn: async () => {
      const res = await httpsCallable(functions, 'getMerchantHomeData')({})
      return res.data as MerchantHomeData
    },
    staleTime: 60_000,
  })
}

export function useMerchantOrders() {
  if (isDevMode()) return { data: { orders: MOCK_MERCHANT_ORDERS }, isLoading: false }
  return useQuery({
    queryKey: ['merchant-orders'],
    queryFn: async () => {
      const res = await httpsCallable(functions, 'getMerchantOrders')({})
      return res.data as { orders: ProductOrder[] }
    },
  })
}
```

---

## UI Changes

### `MerchantHome.tsx`
Replace mock `merchantOrders`, `products` with `useMerchantHome()`:
```tsx
const { recentOrders, recentListings, kpis, isLoading } = useMerchantHome()
```

KPI cards:
- "Commandes en cours" → `kpis.pendingOrders`
- "Total commandé" → `kpis.totalSpentCdf` formatted as CDF
- "Fournisseurs actifs" → count of unique sellerIds in recent listings

Recent orders list (replaces `merchantOrders` mock):
```
Mes commandes récentes
────────────────────────────────────────────
Maïs · 5 000 kg         ⏳ En attente
Commande #8432 · 15 juil.     2 500 000 FC
────────────────────────────────────────────
```

### `MerchantMarket.tsx`
Replace `products` mock with `useProductListings()` (already built in S8-01, filters for `role !== merchant`).
Replace `bourseTicker` with real data from `useProductListings()` (compute ticker from active listings).

Wire `CommanderModal` per SG-07 (call `createProductOrder` CF — remove setTimeout).

### `MerchantBourse.tsx`
Replace `bourseOpportunities` mock with `useProductListings()`.
Replace `bourseTicker` with computed ticker.

Wire `PublierLotModal` and `ReserverLotModal` per SG-07.

**Fix missing mobile `PublierLotModal`**: MobileMerchantBourse currently has no "Publier un lot" button.
Add a FAB (floating action button) at bottom-right:
```tsx
<button
  onClick={() => setPublierOpen(true)}
  className="fixed bottom-20 right-4 z-30 h-14 w-14 bg-purple-700 text-white rounded-full shadow-xl flex items-center justify-center"
>
  <Plus size={24} />
</button>
<PublierLotModal open={publierOpen} onClose={() => setPublierOpen(false)} />
```

---

## Acceptance Criteria
- [ ] `getMerchantHomeData` CF returns recent orders + market listings + KPIs
- [ ] `getMerchantOrders` CF returns merchant's own orders
- [ ] `MerchantHome` KPI cards use real data (not hardcoded)
- [ ] `MerchantHome` recent orders list uses real orders (not mock)
- [ ] `MerchantMarket` product list uses `useProductListings()` (not mock `products`)
- [ ] `MerchantMarket` `CommanderModal` calls `createProductOrder` CF (not setTimeout)
- [ ] `MerchantBourse` listings use `useProductListings()` (not mock `bourseOpportunities`)
- [ ] Mobile `MerchantBourse` has FAB → `PublierLotModal`
- [ ] Dev mode: mock data used across all three screens
