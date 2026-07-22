# SM-06-01 — AgroExchange: place buy/sell order

**Sprint:** SM-06 · AgroExchange  
**Branch:** `feature/sm-06-agro-exchange`

## Context
After listing commodities (SM-06-00), investors and merchants need to place orders. This story implements the order flow.

## Acceptance criteria
- [ ] "Acheter" / "Vendre" opens `AgroOrderModal` (similar to PaymentModal structure)
- [ ] Modal steps: (1) quantity input (kg), (2) review (total price, fees), (3) confirm
- [ ] On confirm: `httpsCallable(functions, "placeAgroOrder")({ listingId, volumeKg, type: "buy" | "sell" })`
- [ ] On success: "Ordre placé!" confirmation + navigate back to list
- [ ] "Créer une annonce" for farmers: `httpsCallable(functions, "createAgroListing")({ commodityName, pricePerKgUsd, availableVolumeTons, region })`
- [ ] Farmer listing form: commodity name (select or text), price/kg, volume (tons), region
- [ ] Active orders visible in `MerchantAgroExchangeScreen` and `FarmerAgroExchangeScreen`
- [ ] `useMyAgroOrders()` hook: `httpsCallable(functions, "getMyAgroOrders")` → `{ orders: AgroOrder[] }`
- [ ] In devMode, operations return mock success; no CF calls made

## Data shape
```ts
interface AgroOrder {
  id: string;
  listingId: string;
  commodityName: string;
  volumeKg: number;
  pricePerKgUsd: number;
  totalUsd: number;
  type: "buy" | "sell";
  status: "pending" | "matched" | "completed" | "cancelled";
  createdAt: { seconds: number };
}
```
