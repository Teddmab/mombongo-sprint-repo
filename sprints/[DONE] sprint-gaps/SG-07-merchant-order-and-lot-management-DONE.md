# SG-07 — Merchant: Wire Action Modals + Order/Lot Tracking

## Why this matters
All four merchant action modals (CommanderModal, ReserverLotModal, PublierLotModal, PreAcheterModal) fire a fake setTimeout and show toast.success. Nothing goes to Firestore. Merchants have no way to track their orders, reservations, or pre-purchases.

## Already implemented (skip these)
- `createProductOrder` CF — ✅ exists in `mombongo-functions/src/merchant/createProductOrder.ts`
- `getMerchantOrders` CF — ✅ exists in `mombongo-functions/src/merchant/getMerchantOrders.ts`
- Both exported from `src/index.ts`

## Remaining work items

### 1. Wire `CommanderModal` to `createProductOrder`
- Replace `setTimeout` stub with `httpsCallable(functions, 'createProductOrder')({ listingId, quantityKg, deliveryAddress, deliveryDate, paymentMethod, notes })`
- Show `orderId` in success screen

### 2. Wire `ReserverLotModal` (reserve bourse lot)
- CF needed: `reserveBourseLot(uid, { opportunityId, lotSizeKg, amountUsd })`
  - Writes to `bourse_investments/{id}` with `investorId: uid, type: 'merchant'`
  - Updates `bourse_opportunities/{opportunityId}.reservedCount += 1`
  - Notifies opportunity creator via push

### 3. Wire `PublierLotModal` (merchant publishes their own lot)
- CF needed: `createBourseOpportunity(uid, { title, commodity, qty, pricePerUnit, currency, availableFrom, province, description })`
  - Writes to `bourse_opportunities/{id}` with `status: 'open'`, `createdBy: uid`

### 4. Wire `PreAcheterModal` (pre-purchase farmer crops)
- CF needed: `createPrePurchase(uid, { farmerId, cropType, qty, pricePerUnitUsd, expectedHarvestDate, notes })`
  - Writes to `pre_purchases/{id}`
  - Notifies farmer via push

### 5. `getMerchantReservations` CF
- Queries `bourse_investments` where `investorId == uid` (merchant reservations)
- Returns with status and opportunity title

### 6. Merchant orders & reservations screen (`MerchantTransactions`)
- New screen accessible from MerchantHome quick actions
- Two tabs: "Commandes" | "Réservations"
- Commandes: calls `getMerchantOrders` — status chips (en attente / confirmée / livrée / annulée)
- Réservations: calls `getMerchantReservations` — status chips
- Tap row → simple detail bottom sheet

### 7. "Mes lots publiés" on MerchantBourse
- Section listing `bourse_opportunities` where `createdBy == uid`
- Status chip per lot (ouvert / complet / clôturé)
- "Retirer" button → calls a `closeBourseLot` CF (soft-delete: status → 'closed')

## Cloud Functions needed (remaining)
- `reserveBourseLot`
- `createBourseOpportunity`
- `createPrePurchase`
- `getMerchantReservations`
- `closeBourseLot` (withdraw a published lot)

## Acceptance criteria
- [ ] CommanderModal creates a real `product_orders` doc via CF
- [ ] ReserverLotModal creates a real `bourse_investments` doc
- [ ] PublierLotModal creates a real `bourse_opportunities` doc visible to others
- [ ] PreAcheterModal creates a real `pre_purchases` doc + notifies farmer
- [ ] Merchant can view their orders and reservations in MerchantTransactions screen
- [ ] "Mes lots publiés" shows on MerchantBourse with withdraw action
