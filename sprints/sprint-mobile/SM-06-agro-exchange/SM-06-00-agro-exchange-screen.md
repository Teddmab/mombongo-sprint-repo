# SM-06-00 — AgroExchange screen: commodity list

**Sprint:** SM-06 · AgroExchange  
**Branch:** `feature/sm-06-agro-exchange`

## Context
The web app has a full AgroExchange feature (agricultural commodity trading). Mobile has no AgroExchange screen. This sprint adds the feature to mobile, matching web parity.

## What web has (reference: `mombongo-web/src/pages/AgroExchangeScreen.tsx`, admin: `mombongo-admin/src/pages/AdminAgroExchange.tsx`)
- Commodity list: name, price per kg, change %, region, available volume
- Role-dispatched: investor/merchant can place buy/sell orders; farmer can list commodities; agent views only
- Price chart per commodity (line chart — last 30 days)

## Acceptance criteria
- [ ] New route: `app/agro-exchange.tsx` → renders `AgroExchangeScreen`
- [ ] `AgroExchangeScreen` dispatches by role (investor, farmer, merchant, agent)
- [ ] `InvestorAgroExchangeScreen`: list of commodities with price + trend; "Acheter" CTA per row
- [ ] `MerchantAgroExchangeScreen`: same list + "Vendre" CTA; my active orders section
- [ ] `FarmerAgroExchangeScreen`: "Mes annonces" + "Créer une annonce" CTA
- [ ] `AgentAgroExchangeScreen`: read-only price list (no trade actions)
- [ ] Accessible from `HomeScreen` quick-action button ("Agro-Bourse")
- [ ] `useAgroExchange()` hook: `httpsCallable(functions, "getAgroExchangeListings")` → `{ listings: Listing[] }`
- [ ] In devMode, returns mock agroExchange listings from `data/mock.ts`

## Data shape
```ts
interface AgroListing {
  id: string;
  commodityName: string;
  pricePerKgUsd: number;
  priceChangePercent: number;
  availableVolumeTons: number;
  region: string;
  sellerId?: string;
  status: "active" | "sold" | "expired";
}
```

## Tab placement
- Not a bottom tab (too many tabs already); accessible via:
  - HomeScreen quick-action card
  - ProfileScreen "Agro-Bourse" menu item
  - Can be added to tabs in SM-06-02 if user testing shows it's high-traffic
