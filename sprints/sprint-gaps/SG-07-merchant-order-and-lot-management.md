# SG-07 — Merchant: Wire Action Modals + Order/Lot Tracking

## Why this matters
All four merchant action modals (CommanderModal, ReserverLotModal, PublierLotModal, PreAcheterModal) fire a fake setTimeout and show toast.success. Nothing goes to Firestore. Merchants have no way to track their orders, reservations, or pre-purchases.

## Current state
- `CommanderModal` — fake 800ms delay, no CF called, no doc created
- `ReserverLotModal` — same
- `PublierLotModal` — same  
- `PreAcheterModal` — same
- No "mes commandes" or "mes réservations" screen for merchants
- Merchant action buttons have no history, no status tracking

## Work items

### 1. Wire `CommanderModal` (order agricultural products)
- CF: `createProductOrder(uid, { productId, qty, deliveryAddress, deliveryDate, paymentMethod, notes })`
  - Writes to `product_orders/{id}` collection
  - Deducts wallet if payment method is wallet (via `processWalletPayment` CF)
  - Notifies the farmer/seller via push (S6-02)
- Return: `{ orderId, status: 'pending' }`

### 2. Wire `ReserverLotModal` (reserve bourse lot)
- CF: `reserveBbourseLot(uid, { opportunityId, lotSize, amountUsd })`
  - Writes to `bourse_investments/{id}` (same collection as investor bourse)
  - Deducts wallet via `processWalletPayment`
  - Updates `bourse_opportunities/{opportunityId}.reservedCount += 1`

### 3. Wire `PublierLotModal` (merchant publishes their own lot)
- CF: `createBourseOpportunity(uid, { title, type, qty, pricePerUnit, currency, availableFrom, province, description })`
  - Writes to `bourse_opportunities/{id}` with `status: 'open'`, `createdBy: uid`

### 4. Wire `PreAcheterModal` (pre-purchase farmer crops)
- CF: `createPrePurchase(uid, { farmerId, cropType, qty, pricePerUnitUsd, expectedHarvestDate, notes })`
  - Writes to `pre_purchases/{id}`
  - Notifies farmer via push (S6-02)

### 5. Merchant orders & reservations screen (`/mes-transactions`)
- New screen accessible from MerchantHome
- Two tabs: "Commandes" | "Réservations"
- Commandes: list from `product_orders` where `buyerId == uid`
  - Status: en attente / confirmée / livrée / annulée
- Réservations: list from `bourse_investments` where `investorId == uid`
  - Status: réservé / confirmé / livré
- Tap row → simple detail modal

### 6. Merchant's published lots
- On MerchantBourse: add a "Mes lots publiés" section
- Lists `bourse_opportunities` where `createdBy == uid`
- Can withdraw/close a lot they published

## Cloud Functions needed
- `createProductOrder(data)` → `product_orders`
- `reserveBbourseLot(data)` → `bourse_investments`
- `createBourseOpportunity(data)` → `bourse_opportunities`
- `createPrePurchase(data)` → `pre_purchases`
- `getMerchantOrders(uid)` → product orders where buyerId == uid
- `getMerchantReservations(uid)` → bourse investments where investorId == uid

## Acceptance criteria
- [ ] CommanderModal creates a real `product_orders` doc
- [ ] ReserverLotModal creates a real `bourse_investments` doc + deducts wallet
- [ ] PublierLotModal creates a real `bourse_opportunities` doc visible to others
- [ ] PreAcheterModal creates a real `pre_purchases` doc + notifies farmer
- [ ] Merchant can view their orders and reservations in a dedicated screen
