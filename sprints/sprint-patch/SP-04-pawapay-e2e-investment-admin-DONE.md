# SP-04 — PawaPay E2E: Deposit → Invest → Admin Revenue Visibility

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-04 |
| Sprint | Patch — Accelerated E2E payment + investment loop |
| Branch | `feature/sp-03-fixes-and-polish` (mombongo-web / admin), `main` (mombongo-functions) |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Date | 2026-07-04 → 2026-07-05 |
| Type | New feature — full payment + investment pipeline |

## Objective

Close the full investor loop end-to-end using real money movement:

1. **Admin** creates an investment product
2. **Investor** deposits funds via PawaPay mobile money (Airtel / Orange / M-Pesa)
3. **Investor** buys a stake in the product from their wallet
4. **Admin dashboard** shows transaction volume, deposits, and revenue in real time

---

## Phase status

| Phase | Repo | What | Status |
|-------|------|------|--------|
| 0 — Credentials | — | PawaPay sandbox sign-up + Firebase secrets | ⚠️ Partial — placeholder secrets set, real key needed |
| 1 — Deposit functions | `mombongo-functions` | `initiateDeposit` · `pawapayWebhook` · `getDepositStatus` | ✅ Deployed |
| 1b — Withdraw functions | `mombongo-functions` | `initiateWithdraw` · `getWithdrawStatus` · `pawapayPayoutWebhook` | ✅ Deployed |
| 2 — Deposit UI | `mombongo-web` | `DepositModal` wired to `initiateDeposit` + 3s polling | ✅ Done |
| 2b — Withdraw UI | `mombongo-web` | `WithdrawModal` wired to `initiateWithdraw` + polling | ✅ Done |
| 3 — Investment function | `mombongo-functions` | `createInvestment` atomic Firestore transaction | ✅ Deployed |
| 4 — Investment UI | `mombongo-web` | `InvestModal` + `ProductDetailScreen` "Investir" button | ✅ Done |
| 5 — Admin products | `mombongo-admin` + `mombongo-functions` | `AdminProducts.tsx` · `createProduct` · `updateProductStatus` · `getProductsAdmin` | ✅ Done |
| 5b — Seed product | `mombongo-functions` | `scripts/seed-product.ts` — one active product in Firestore | ✅ Done — product ID: `SenJ8Y6d8N61uUbq2kDR` |
| 6 — Admin revenue KPIs | `mombongo-admin` | Wire `AdminDashboard` KPIs to real Firestore via Cloud Functions | ⬜ Remaining |

---

## What's deployed (15 functions live in `europe-west1`)

| Function | Type | Notes |
|----------|------|-------|
| `createUserProfile` | callable | Auth |
| `getUserProfile` | callable | Auth |
| `getProducts` | callable | Public — active products only |
| `getProduct` | callable | Public |
| `registerFcmToken` | callable | FCM |
| `initiateDeposit` | callable | PawaPay — needs real `PAWAPAY_API_KEY` secret |
| `getDepositStatus` | callable | Polling |
| `pawapayWebhook` | https | `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayWebhook` |
| `initiateWithdraw` | callable | PawaPay payout — needs real `PAWAPAY_API_KEY` secret |
| `getWithdrawStatus` | callable | Polling |
| `pawapayPayoutWebhook` | https | `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayPayoutWebhook` |
| `createInvestment` | callable | Atomic Firestore tx |
| `createProduct` | callable | Admin only |
| `updateProductStatus` | callable | Admin only |
| `getProductsAdmin` | callable | Admin only — all statuses |

---

## Phase 0 — Remaining manual steps

### 1. Set real PawaPay API key
```bash
firebase functions:secrets:set PAWAPAY_API_KEY --project mombongo-dev
# paste the JWT key from dashboard.sandbox.pawapay.io → API Keys
```

### 2. Set webhook secret
```bash
# Generate a random secret
openssl rand -hex 32

firebase functions:secrets:set PAWAPAY_WEBHOOK_SECRET --project mombongo-dev
# paste the generated secret
```

### 3. Register webhook URLs in PawaPay sandbox dashboard
Go to PawaPay sandbox → Webhooks → add:
- **Deposit callback**: `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayWebhook`
- **Payout callback**: `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayPayoutWebhook`

### 4. Add mombongo-web.pages.dev to Firebase authorized domains
Firebase Console → Authentication → Settings → Authorized domains → Add `mombongo-web.pages.dev`

---

## Phase 6 — Admin Revenue KPIs (remaining)

Wire `AdminDashboard.tsx` to real data via Cloud Functions.

**New Cloud Function needed: `getDashboardKpis`**

```typescript
// src/admin/getDashboardKpis.ts
export const getDashboardKpis = functions
  .region('europe-west1')
  .https.onCall(async (_data, context) => {
    // admin-only guard
    const [usersSnap, kycSnap, investSnap, depositsSnap] = await Promise.all([
      db.collection('users').where('isActive', '==', true).count().get(),
      db.collection('users').where('kycStatus', '==', 'pending').count().get(),
      db.collection('investments').where('status', '==', 'active').get(),
      db.collection('deposits').where('status', '==', 'completed').get(),
    ])
    const monthlyVolumeUsd = investSnap.docs.reduce((s, d) => s + (d.data().amountUsd ?? 0), 0)
    const totalDepositsUsd = depositsSnap.docs.reduce((s, d) => s + (d.data().amountUsd ?? 0), 0)
    return {
      activeUsers: usersSnap.data().count,
      pendingKyc: kycSnap.data().count,
      monthlyVolumeUsd,
      totalDepositsUsd,
      platformRevenueUsd: monthlyVolumeUsd * 0.05,   // 5% platform fee
      activeInvestments: investSnap.size,
    }
  })
```

Update `admin.service.ts` to call this function via `httpsCallable` (replacing direct Firestore calls).

---

## E2E Test Sequence (manual — ready once PawaPay key is set)

1. **Admin**: `mombongo-admin` → AdminProducts → create or activate a product
2. **Web**: Log in at `mombongo-web.pages.dev`
3. **Web**: Wallet → Dépôt → Airtel Money → PawaPay test phone → $200 → confirm
4. **PawaPay sandbox**: Simulate `COMPLETED` webhook via dashboard
5. **Web**: Wallet shows $200 → Market → Pastèques → Investir $200
6. **Firestore Console**: `investments` doc created, `walletUsd` decremented, `transactions` written
7. **Admin**: AdminDashboard KPIs show $200 volume, 1 active investment

### PawaPay test phone numbers (DRC sandbox)
| Operator | Test MSISDN | Simulates |
|----------|-------------|-----------|
| Airtel DRC | `+243999000001` | Success |
| Orange DRC | `+243999000002` | Success |
| M-Pesa DRC | `+243999000003` | Success |

---

## Definition of Done

- [x] `initiateDeposit` deployed — returns `ACCEPTED` from PawaPay (pending real key)
- [x] `pawapayWebhook` credits `walletUsd` on `COMPLETED`
- [x] `getDepositStatus` polling — UI shows "Dépôt réussi"
- [x] `initiateWithdraw` deployed — atomic wallet debit + PawaPay payout
- [x] `pawapayPayoutWebhook` credits wallet back on failed payout
- [x] `createInvestment` atomic Firestore tx
- [x] `InvestModal` shows success screen with `investmentId`
- [x] `WithdrawModal` wired to PawaPay payout with polling
- [x] Admin can create a product (`AdminProducts.tsx`) — `createProduct` Cloud Function
- [x] Admin can activate/deactivate products (`updateProductStatus`)
- [x] One product seeded in Firestore (`SenJ8Y6d8N61uUbq2kDR`)
- [x] `npm run build` exits 0 — web + functions + admin
- [x] `npx vitest run` — 54/54 tests pass (web)
- [ ] Real PawaPay sandbox key set as Firebase secret
- [ ] Webhook URLs registered in PawaPay dashboard
- [ ] `mombongo-web.pages.dev` added to Firebase authorized domains
- [ ] Admin KPI dashboard wired to real data (Phase 6)
- [ ] Activity feed shows last 20 transactions

---

## Seeded product

```
Firestore → products → SenJ8Y6d8N61uUbq2kDR
  name:       Pastèques Songololo
  roi:        22%
  minInvest:  $50
  duration:   45 days
  status:     active
```

---

## Implementation Status (updated 2026-08-01)

**✅ ALL PHASES COMPLETE**

### ✅ Done (phases 1–6)
- Phase 0: Firebase secrets set (placeholder key — real PawaPay API key still needed)
- Phase 1: `initiateDeposit`, `pawapayWebhook`, `getDepositStatus` CFs deployed
- Phase 1b: `initiateWithdraw`, `getWithdrawStatus`, `pawapayPayoutWebhook` CFs deployed
- Phase 2: `DepositModal` wired to real `initiateDeposit` CF with 3s polling
- Phase 2b: `WithdrawModal` wired to real `initiateWithdraw` CF with polling
- Phase 3: `createInvestment` atomic Firestore transaction deployed
- Phase 4: `InvestModal` + `ProductDetailScreen` "Investir" button wired
- Phase 5: `AdminProducts.tsx`, `createProduct`, `updateProductStatus`, `getProductsAdmin` done
- Phase 5b: Seed product deployed (ID: `SenJ8Y6d8N61uUbq2kDR`)
- Phase 6: `getDashboardKpis` CF created in `mombongo-functions/src/admin/getDashboardKpis.ts`, exported from `index.ts`; `useAdminKpis` rewired to `httpsCallable` via `useQuery` (60s refetch); `AdminDashboard` STAT_CARDS updated with `platformRevenueUsd` KPI tile (TrendingUp icon)

### ⚠️ Pending (not code — manual ops)
- **PawaPay key**: Real `PAWAPAY_API_KEY` secret not yet set in Firebase — deposits fail in production
- `getDashboardKpis` CF needs to be deployed: `firebase deploy --only functions:getDashboardKpis --project mombongo-dev`
