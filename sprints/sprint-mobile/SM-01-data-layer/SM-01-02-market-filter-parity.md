# SM-01-02 — Market filter parity with web

**Sprint:** SM-01 · Data layer  
**Branch:** `feature/sm-01-data-layer`

## Context
Web's InvestorMarketScreen has a `FilterSheet` with filters: category, crop type, region, harvest month, and sort order. Mobile's `InvestorMarketScreen` has a simpler search bar but may be missing the advanced filters. This story brings mobile to parity.

## Acceptance criteria
- [ ] `InvestorMarketScreen` has a filter button opening a bottom sheet
- [ ] Bottom sheet filters: Category (chips), Crop type (chips), Region (chips), Sort (Rendement / Horizon / Popularité)
- [ ] Active filters shown as chips below the search bar (with × to clear)
- [ ] Filtered count shown: "N produits"
- [ ] Filters persist while sheet is open; applied on "Appliquer" press
- [ ] FarmerMarketScreen, AgentMarketScreen have the same filter bottom sheet
- [ ] MerchantMarketScreen has: search + category filter (buy vs sell context)

## Implementation notes
- Use `BottomSheetShell` component (already exists at `components/ui/BottomSheetShell.tsx`)
- Filter state: local `useState` in each screen (no global state needed)
- Filter data: categories and regions from `useProducts()` — derive unique values
