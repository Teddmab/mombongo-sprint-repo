# SU-03-01 — Cumulative revenue counter on farmer dashboard

**Sprint:** SU-03 · Outcomes & growth  
**Branch:** `feature/su-03-outcomes-growth`  
**Effort:** ~4 days

## Context
The farmer has no visible proof that Mombongo is delivering on its promise. This story adds a "Revenus via Mombongo" widget to the dashboard showing total earnings generated through the platform — from completed Bourse sales and disbursed loans. Seeing this number grow is the strongest retention mechanic possible.

## What counts as "revenue via Mombongo"
- Completed Bourse transaction (seller side): `transactions/{id}.sellerAmount` where `status === 'completed'`
- Disbursed credit: `financingApplications/{id}.disbursedAmount` where `status === 'disbursed'`
- Pre-purchase payment received: `preAchatContracts/{id}.amountPaidCdf` where `status === 'completed'`

## Implementation

### CF: update `calculateFarmerRevenue(uid)` 
- Triggered on writes to `transactions`, `financingApplications`, `preAchatContracts`
- Aggregates all qualifying amounts → stores `users/{uid}.cumulativeRevenueCdf: number`
- Also computes `thisMonthRevenueCdf` for monthly breakdown
- isDevMode mock: 185,000 FC cumulative, 42,000 FC this month

### New hook: `useFarmerRevenue` (`src/hooks/useFarmerRevenue.ts`)
```typescript
// Returns { cumulativeCdf: number, thisMonthCdf: number, isLoading: boolean }
// isDevMode() → { cumulativeCdf: 185_000, thisMonthCdf: 42_000, isLoading: false }
```

### UI: `RevenueWidget` on `AgricultorHome.tsx`
- Placed below the DashboardCTACard (SU-01-02), above the 4 quick actions
- Shows: "Vos gains Mombongo" with large CDF amount and "ce mois" sub-figure
- Animated count-up from 0 on first render (respects `data-slow` — instant if slow)
- If cumulativeRevenue === 0: shows "Effectuez votre première vente pour commencer à suivre vos revenus →"
- Currency format: `xxx,xxx FC` (Congolese Franc convention — thousands separator = `,`)

## Acceptance criteria
- [ ] `RevenueWidget` renders on farmer dashboard (mobile + desktop)
- [ ] When no sales: zero-state with CTA to Bourse
- [ ] After a completed transaction: widget updates within 60 sec (CF trigger)
- [ ] Count-up animation plays on first render; is instant on `data-slow`
- [ ] `thisMonthCdf` resets correctly at month boundary
- [ ] `isDevMode()` returns 185,000 FC without CF call

## Smoke test steps
1. Log in as farmer with no sales → verify zero-state widget with "Effectuez votre première vente" CTA
2. Complete a Bourse transaction (or set `transactions/{id}.status = 'completed'` in Firestore) → wait 60 sec → refresh → verify amount updated
3. Verify currency format: "185,000 FC" (comma as thousands separator)
4. Slow connection mode → verify no count-up animation, number appears instantly
5. Check month boundary (set `thisMonthRevenueCdf` manually) → verify reset
