# SU-03-02 — "Pour vous" tab on Bourse (buyer matching for farmers)

**Sprint:** SU-03 · Outcomes & growth  
**Branch:** `feature/su-03-outcomes-growth`  
**Effort:** ~5 days

## Context
The Bourse screen shows a flat list of all listings. Farmers have no way to see which buyers are actively looking for their specific crop in their province. This story adds a "Pour vous" tab that surfaces matching demand — creating urgency and showing the farmer that the market is live.

## Matching logic

A buyer opportunity appears in "Pour vous" if:
- `order.commodityRequested` matches farmer's `primaryCrop` (case-insensitive)
- `order.province` matches farmer's province OR `order.deliveryFlexible === true`
- `order.status === 'open'` (buyer still looking)
- `order.quantityKgNeeded <= farmer's available stock * 1.2` (within 20% of their capacity)

Sorted by: `order.createdAt DESC` (most recent buyer first)

## Implementation

### CF: `getBourseBuyerMatches()` → `{ matches: BuyerMatch[], totalActive: number }`
```typescript
type BuyerMatch = {
  orderId: string
  buyerName: string            // merchant name
  commodityRequested: string
  quantityKgNeeded: number
  priceOfferedCdf: number | null   // null if buyer wants to negotiate
  province: string
  deliveryFlexible: boolean
  createdAt: Timestamp
  expiresAt: Timestamp | null
}
```

### New hook: `useBourseBuyerMatches` (`src/hooks/useBourseBuyerMatches.ts`)
- `isDevMode()` → returns 3 mock buyer matches for Maïs in Kongo Central
- Real path: `httpsCallable(functions, 'getBourseBuyerMatches')()`

### UI: Add "Pour vous" tab to `AgricultorBourse` screen

Current tabs (if any) or add tab navigation:
- **Mes annonces** — existing listings view
- **Pour vous** — new buyer matches view (badge with count: "Pour vous ·  3")

"Pour vous" tab content:
- Header: "3 acheteurs cherchent [crop] dans votre région"
- List of BuyerMatch cards: buyer name, quantity wanted, price offered (or "Prix à négocier"), province, time since posted
- CTA on each card: "Contacter cet acheteur →" → opens a message/contact modal (or routes to existing commander flow)
- Empty state: "Aucun acheteur actif pour votre culture en ce moment. Publiez votre annonce pour être visible."

## Acceptance criteria
- [ ] "Pour vous" tab visible on Bourse screen for farmer role
- [ ] Tab badge shows count of active matches
- [ ] Match cards show correct buyer details
- [ ] "Contacter" CTA navigates to correct flow
- [ ] Empty state renders when no matches found
- [ ] Tab does NOT render for merchant/investor roles
- [ ] `isDevMode()` returns 3 mock matches

## Smoke test steps
1. Log in as farmer (primaryCrop = Maïs, province = Kongo Central) → open Bourse → verify "Pour vous" tab
2. Tab badge shows "Pour vous · 3" (mock data)
3. Tap a match card → verify contact/order flow opens
4. Empty state: change isDevMode mock to return [] → verify empty state message
5. Log in as merchant → verify "Pour vous" tab NOT visible
