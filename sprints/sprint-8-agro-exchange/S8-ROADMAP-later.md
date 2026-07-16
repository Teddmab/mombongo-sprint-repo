# Sprint 8 — Agro Exchange — Later-Phase Roadmap (S9–S12)

This document outlines the sprints that follow S8-00 through S8-03 but are **not immediate priorities**. They are defined enough to plan against, but implementation should start after S8 is fully shipped and validated.

---

## Sprint 9 — Market Intelligence & Province Price Index

**Goal**: Give all actors real price information by province so they can make informed decisions.

### S9-01 — Province Price Map & Daily Index
- Admin enters or import daily prices per commodity per province (from S8-00's `bourse_prices_by_province`)
- A scheduled CF (`scheduleDailyPriceAggregate`) computes daily averages from completed `bourse_contracts` and writes to `bourse_prices_by_province`
- New screen or tab: **Marchés — Prix par province** (map + table)
  - Select commodity → see all provinces with price heat-map
  - Province drill-down: price history chart (recharts AreaChart, 30-day window)
  - Price badge shows % change vs yesterday
- **Estimate**: 4h (functions: 2h, web: 2h)

### S9-02 — Market Analytics Dashboard
- A dedicated "Tendances du marché" page visible to all roles:
  - Volume traded per commodity this week (bar chart)
  - Top 3 commodities by price increase
  - Peak demand calendar (months with most buyer orders per commodity)
  - Transaction count and total value (CDF) for the current month
- Powered by a `getMarketStats` CF that aggregates over `bourse_contracts` and `buyer_orders`
- **Estimate**: 3h

### S9-03 — Price Alerts
- Farmer/seller can set a price alert: "Notify me when Maïs in Kinshasa reaches 450 FC/kg"
- Stored in `price_alerts` collection
- Scheduled CF checks latest prices against alerts → sends FCM push (requires S6-02 FCM first)
- **Estimate**: 3h (depends on S6-02 FCM)

---

## Sprint 10 — Logistics & Certified Warehouses

**Goal**: Connect the commodity exchange to physical logistics and warehouse storage, completing the supply chain.

### S10-01 — Logistics Partner Integration (Stub)
- After contract is active, seller can book a transport slot from a list of registered carriers
- `transport_partners` collection (seeded by admin: carrier name, route coverage, rate per km)
- `createShipmentBooking` CF: records booking, updates contract with `transportPartnerId`
- Web: simple "Choisir un transporteur" step in ContractModal after escrow is funded
- **Estimate**: 4h

### S10-02 — Certified Warehouse Receipts
- Farmers can deposit produce at a certified warehouse (partner) and receive a digital receipt
- `warehouse_receipts` collection:
  ```
  { id, farmerId, warehouseId, commodity, quantityKg, quality, depositedAt, expiresAt, status: 'active'|'redeemed' }
  ```
- `createWarehouseReceipt` CF (admin-only via `mombongo-admin` initially)
- Receipt shown in farmer's profile with QR code (base64 encoded)
- Receipt can be presented as collateral for Financement applications (S4-xx link)
- **Estimate**: 4h

### S10-03 — Delivery Tracking (Simple)
- After `confirmShipment`, contract shows estimated delivery date + carrier contact
- Seller can update shipment status: `shipped` → `in_transit` → `arrived`
- `updateShipmentStatus` CF: validates uid == sellerId, updates contract
- Buyer receives FCM notification when status changes (depends on S6-02)
- Full GPS tracking is a future integration (logistics partner API)
- **Estimate**: 3h

---

## Sprint 11 — Advanced Exchange Features

**Goal**: Add mechanisms that make the Bourse more liquid and competitive.

### S11-01 — Auction Mechanism
- Seller can choose "enchère" mode instead of fixed price when creating a listing
- `listing.mode: 'fixed' | 'auction'`
- `auction_bids` sub-collection on `product_listings`
- Auction has `endTime`; highest bid wins at close; `auctionCloseCF` runs on schedule
- Web: bid UI on listing detail, countdown timer, live bid feed (poll every 30s)
- **Estimate**: 6h

### S11-02 — Agricultural Insurance Integration (Stub)
- Partner page showing available micro-insurance products (seeded data)
- "Souscrire" opens a form → sends data to insurance partner API (placeholder)
- Initial implementation: form submission → email/webhook to partner (no real API integration yet)
- **Estimate**: 3h

### S11-03 — Group Input Purchasing (Intrants Groupés)
- Farmers can join a "groupe d'achat" for seeds, fertilizers, tools
- `input_purchase_groups` collection: commodity input, target quantity, current participants, deadline, supplierPrice
- `joinInputGroup` CF: adds user to group
- Admin creates groups; when target quantity reached, group is "confirmed"
- Web: new sub-tab in Academia or Bourse showing open input groups
- **Estimate**: 4h

### S11-04 — Supply Contract Market (Forward Contracts)
- Agribusiness companies can post a supply need with a future delivery date and guaranteed price
- `supply_contracts` collection: company, commodity, quantity, pricePerKgCdf, deliveryDate, deadline
- Farmers browse and commit to fulfill a contract (partial fulfillment allowed)
- Distinct from spot market (S8-xx): these are future delivery contracts
- Payment on delivery, not escrow upfront (risk model TBD with legal)
- **Estimate**: 5h

---

## Sprint 12 — Mombongo Agro Exchange Vision

**Goal**: Establish the Mombongo Agro Exchange brand as a national price reference and intelligence platform.

### S12-01 — DRC Agricultural Price Index (MAE Index)
- Weighted average of completed transaction prices across all provinces → single daily index value per commodity
- `computeMaeIndex` scheduled CF (runs daily at 23:00)
- `mae_index` collection: `{ commodity, date, indexValue, previousValue, changePercent, transactionCount, totalVolumeKg }`
- Web: dedicated index page showing the MAE index for each commodity, historical chart, methodology note
- Can be published as a public API endpoint for press/research
- **Estimate**: 4h

### S12-02 — National Availability Map
- Interactive map of DRC showing which territories have active product listings
- Territory-level dots with commodity counts and average price
- Powered by `getAvailabilityMap` CF aggregating `product_listings` by territory + commodity
- Map rendered with SVG (custom DRC territory SVG) or a mapbox-lite equivalent
- **Estimate**: 6h (map rendering is complex, consider external lib with data URI)

### S12-03 — Harvest Forecast Module
- Farmers enter expected harvest: commodity, expected quantity, expected date, province
- `harvest_forecasts` collection: `{ farmerId, commodity, expectedQtyKg, expectedDate, province, createdAt }`
- `createHarvestForecast` CF (simple write)
- Aggregated view in admin: expected supply by commodity + month → helps buyers plan ahead
- Public-facing "Prévisions de récoltes" page showing aggregate forecast (without personal data)
- **Estimate**: 3h

### S12-04 — Cooperative Role Support
- Add `cooperative` as a formal role (distinct from farmer)
- Cooperative can list produce on behalf of multiple member farmers (linked `memberFarmerIds[]`)
- Cooperative members can see collective listings in their bourse view
- `createCooperativeListing` CF: validates that caller is a cooperative role, links to members
- **Estimate**: 4h

---

## Priority Matrix

| Sprint | Do now? | Reason |
|--------|---------|--------|
| S8-00 | Yes | Data model must come first |
| S8-01 | Yes | Core: farmers can't sell without this |
| S8-02 | Yes | Core: buyers can't buy without this |
| S8-03 | Yes | Core: no trust without escrow |
| S9-01 | After S8 | High value for all actors — price reference |
| S9-02 | After S8 | Nice-to-have dashboard |
| S9-03 | After S6-02 | Depends on FCM |
| S10-01 | After S8 | Good for closing the loop |
| S10-02 | After S10-01 | Warehouse receipts need logistics first |
| S10-03 | After S8-03 | Delivery tracking is part of the escrow flow |
| S11-xx | Phase 3 | Advanced features, not blocking MVP |
| S12-xx | Phase 4 | Vision features, significant investment |

---

## Dependency on Existing S3-xx Sprints

The S3-xx sprints (logistics investment model) are **not replaced** by S8-xx. They should still be implemented first because:
1. S3-00 establishes the functions project pattern
2. S3-01/02 wire the existing Bourse UI to live data
3. S3-03 adds the investor investment flow in CDF

S8-xx then adds the commodity marketplace as a second tab/section within the same Bourse screen. The investor view of Bourse will have:
- Tab 1: "Investir" → S3-xx logistics investment opportunities
- Tab 2: "Marché" → S8-xx commodity exchange listings + orders

The data flows do not conflict.
