# SP-04 — PawaPay E2E: Deposit → Invest → Admin Revenue Visibility

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-04 |
| Sprint | Patch — Accelerated E2E payment + investment loop |
| Branch | `feature/sp-04-pawapay-e2e` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Date | 2026-07-04 |
| Type | New feature — full payment + investment pipeline |

## Objective

Close the full investor loop end-to-end using real money movement:

1. **Admin** creates an investment product (e.g. "Pastèques de Songololo")
2. **Investor** deposits funds via PawaPay mobile money (Airtel / Orange / M-Pesa)
3. **Investor** buys a stake in the product from their wallet
4. **Admin dashboard** shows transaction volume, deposits, and revenue in real time

This sprint intentionally scopes to PawaPay only (no Stripe card) so the full loop can be validated quickly. Stripe card payments follow in a later sprint.

---

## Repos and phases

| Phase | Repo | What |
|-------|------|------|
| 0 — Credentials | — | PawaPay sandbox sign-up + Firebase secrets |
| 1 — Deposit (PawaPay) | `mombongo-functions` | `initiateDeposit` · `pawapayWebhook` · `getDepositStatus` |
| 2 — Deposit UI | `mombongo-web` | Wire `DepositModal` mobile money tab to real function |
| 3 — Investment | `mombongo-functions` | `createInvestment` atomic Firestore transaction |
| 4 — Investment UI | `mombongo-web` | `InvestModal` + wire ProductDetailScreen "Invest" button |
| 5 — Admin products | `mombongo-admin` | `AdminProducts.tsx` — create / activate / deactivate products |
| 6 — Admin revenue | `mombongo-admin` | Wire `AdminDashboard` KPIs + bar chart to real Firestore |

---

## Phase 0 — Credentials (manual, ~15 min)

### PawaPay sandbox
1. Sign up at https://dashboard.sandbox.pawapay.io/
2. Create a project, copy the **Bearer API key**
3. Note the test phone numbers for Airtel DRC, Orange DRC, M-Pesa DRC (listed in PawaPay docs)

### Firebase secrets
```bash
cd mombongo-functions
firebase functions:secrets:set PAWAPAY_API_KEY --project mombongo-dev
# paste the API key when prompted

firebase functions:secrets:set PAWAPAY_WEBHOOK_SECRET --project mombongo-dev
# create any random string for sandbox; use PawaPay dashboard value in production
```

### Webhook URL (after Phase 1 deploy)
Register in PawaPay sandbox dashboard:
```
https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayWebhook
```

---

## Phase 1 — PawaPay Cloud Functions (`mombongo-functions`)

> Full code specs already written in `S6-01-payments.md`. Reproduce here for this branch.

### File structure
```
mombongo-functions/src/
  index.ts                      ← export all functions
  payments/
    initiateDeposit.ts          ← onCall — mobile money deposit via PawaPay
    pawapayWebhook.ts           ← onRequest — PawaPay confirms deposit
    getDepositStatus.ts         ← onCall — poll deposit status (no onSnapshot from frontend)
    initiateWithdraw.ts         ← onCall — PawaPay payout (phase 1b)
```

### `initiateDeposit` (onCall)

```typescript
// src/payments/initiateDeposit.ts
import * as functions from 'firebase-functions'
import * as admin from 'firebase-admin'
import axios from 'axios'
import { v4 as uuid } from 'uuid'

const db = admin.firestore()
const PAWAPAY_BASE = 'https://api.sandbox.pawapay.io'  // switch to api.pawapay.io in prod

export const initiateDeposit = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const { amountUsd, phone, operator } = data as {
      amountUsd: number
      phone: string
      operator: string   // 'AIRTEL_DRC' | 'ORANGE_DRC' | 'MPESA_DRC'
    }

    if (!amountUsd || amountUsd < 5)
      throw new functions.https.HttpsError('invalid-argument', 'Minimum deposit $5')
    if (!phone || !operator)
      throw new functions.https.HttpsError('invalid-argument', 'phone and operator required')

    const depositId = uuid()
    const apiKey = process.env.PAWAPAY_API_KEY

    await db.collection('deposits').doc(depositId).set({
      userId: uid, depositId, amountUsd,
      currency: 'USD',
      phone, operator,
      status: 'pending',
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    })

    const response = await axios.post(
      `${PAWAPAY_BASE}/deposits`,
      {
        depositId,
        amount: String(amountUsd),
        currency: 'USD',
        correspondent: operator,
        payer: { type: 'MSISDN', address: { value: phone } },
        statementDescription: 'Dépôt Mombongo',
      },
      { headers: { Authorization: `Bearer ${apiKey}` } }
    )

    if (response.data.status !== 'ACCEPTED') {
      await db.collection('deposits').doc(depositId).update({ status: 'failed' })
      throw new functions.https.HttpsError(
        'internal',
        `PawaPay rejected: ${response.data.rejectionReason?.rejectionCode ?? 'unknown'}`
      )
    }

    return { depositId, status: 'ACCEPTED' }
  })
```

### `getDepositStatus` (onCall)

Frontend polls this every 3 s instead of using `onSnapshot` (no Firestore direct access from web).

```typescript
// src/payments/getDepositStatus.ts
export const getDepositStatus = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const { depositId } = data as { depositId: string }
    const snap = await db.collection('deposits').doc(depositId).get()
    if (!snap.exists || snap.data()?.userId !== uid)
      throw new functions.https.HttpsError('not-found', 'Deposit not found')

    return { status: snap.data()?.status, amountUsd: snap.data()?.amountUsd }
  })
```

### `pawapayWebhook` (onRequest — HTTP)

```typescript
// src/payments/pawapayWebhook.ts
import crypto from 'crypto'

export const pawapayWebhook = functions
  .region('europe-west1')
  .https.onRequest(async (req, res) => {
    // Verify signature
    const signature = req.headers['x-pawapay-signature'] as string
    const expected = crypto
      .createHmac('sha256', process.env.PAWAPAY_WEBHOOK_SECRET!)
      .update(JSON.stringify(req.body))
      .digest('hex')
    if (signature !== expected) { res.status(401).send('Invalid signature'); return }

    const { depositId, status } = req.body

    if (status !== 'COMPLETED') {
      await db.collection('deposits').doc(depositId).update({ status: 'failed' })
      res.status(200).send('OK'); return
    }

    const depositRef = db.collection('deposits').doc(depositId)
    const depositSnap = await depositRef.get()
    if (!depositSnap.exists || depositSnap.data()?.status !== 'pending') {
      res.status(200).send('Already processed'); return
    }

    const { userId, amountUsd } = depositSnap.data()!
    const now = admin.firestore.FieldValue.serverTimestamp()

    await db.runTransaction(async tx => {
      tx.update(db.collection('users').doc(userId), {
        walletUsd: admin.firestore.FieldValue.increment(amountUsd),
      })
      tx.update(depositRef, { status: 'completed', completedAt: now })
      tx.set(db.collection('transactions').doc(), {
        userId, type: 'deposit', method: 'mobile_money',
        amountUsd, currency: 'USD', status: 'completed',
        pawapayDepositId: depositId, createdAt: now,
      })
    })

    res.status(200).send('OK')
  })
```

### Export in `src/index.ts`

```typescript
export { initiateDeposit }   from './payments/initiateDeposit'
export { getDepositStatus }  from './payments/getDepositStatus'
export { pawapayWebhook }    from './payments/pawapayWebhook'
export { initiateWithdraw }  from './payments/initiateWithdraw'  // phase 1b
```

### Deploy

```bash
cd mombongo-functions
npm install axios uuid
npm run build
firebase deploy --only functions:initiateDeposit,getDepositStatus,pawapayWebhook --project mombongo-dev
```

---

## Phase 2 — Wire DepositModal (`mombongo-web`)

Replace the `setTimeout` fake-submit in `WalletModals.tsx` (`DepositModal` component) with real PawaPay calls.

```typescript
// In DepositModal — mobile money submit handler
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { useQuery } from '@tanstack/react-query'

const initiateDeposit = httpsCallable<object, { depositId: string; status: string }>(
  functions, 'initiateDeposit'
)
const getDepositStatus = httpsCallable<{ depositId: string }, { status: string; amountUsd: number }>(
  functions, 'getDepositStatus'
)

// Step 1: initiate
async function handleMobileMoneySubmit() {
  setLoading(true)
  try {
    const { data } = await initiateDeposit({ amountUsd: amount, phone, operator })
    setDepositId(data.depositId)
    setStep('waiting')  // "STK push envoyé — confirmez sur votre téléphone"
  } catch (err: any) {
    toast.error(err.message)
  } finally {
    setLoading(false)
  }
}

// Step 2: poll until completed
const { data: statusData } = useQuery({
  queryKey: ['depositStatus', depositId],
  queryFn: () => getDepositStatus({ depositId: depositId! }).then(r => r.data),
  enabled: !!depositId && step === 'waiting',
  refetchInterval: (data) => (data?.status === 'completed' ? false : 3000),
})

useEffect(() => {
  if (statusData?.status === 'completed') {
    queryClient.invalidateQueries({ queryKey: ['userProfile'] })
    setStep('success')
  }
}, [statusData?.status])
```

**Operator mapping constants** (add to `src/lib/constants.ts`):
```typescript
export const MOBILE_OPERATORS = [
  { id: 'AIRTEL_DRC',  label: 'Airtel Money', color: '#e8000d' },
  { id: 'ORANGE_DRC',  label: 'Orange Money', color: '#ff6600' },
  { id: 'MPESA_DRC',   label: 'M-Pesa',       color: '#00a651' },
]
```

---

## Phase 3 — `createInvestment` Cloud Function (`mombongo-functions`)

> Full spec in `S2-04-investment-flow.md`. Deploy after Phase 1.

```typescript
// src/investments/createInvestment.ts
export const createInvestment = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const { productId, amountUsd } = data as { productId: string; amountUsd: number }

    const investmentRef = db.collection('investments').doc()
    const txRef = db.collection('transactions').doc()

    await db.runTransaction(async tx => {
      const [userSnap, productSnap] = await Promise.all([
        tx.get(db.collection('users').doc(uid)),
        tx.get(db.collection('products').doc(productId)),
      ])

      if (!userSnap.exists) throw new functions.https.HttpsError('not-found', 'User not found')
      if (!productSnap.exists || productSnap.data()?.status !== 'active')
        throw new functions.https.HttpsError('not-found', 'Product not available')

      const walletUsd: number = userSnap.data()?.walletUsd ?? 0
      const minInvest: number = productSnap.data()?.minInvest ?? 0

      if (amountUsd < minInvest)
        throw new functions.https.HttpsError('invalid-argument', `Minimum investment is $${minInvest}`)
      if (walletUsd < amountUsd)
        throw new functions.https.HttpsError('failed-precondition', 'Insufficient wallet balance')

      const now = admin.firestore.FieldValue.serverTimestamp()
      const harvestDate = new Date()
      harvestDate.setDate(harvestDate.getDate() + (productSnap.data()?.duration ?? 30))

      tx.set(investmentRef, {
        investorId: uid, productId, amountUsd,
        roi: productSnap.data()?.roi,
        status: 'active',
        harvestDate: admin.firestore.Timestamp.fromDate(harvestDate),
        investedAt: now,
        productName: productSnap.data()?.name,
        productIcon: productSnap.data()?.icon,
      })
      tx.update(db.collection('users').doc(uid), {
        walletUsd: admin.firestore.FieldValue.increment(-amountUsd),
        totalInvestedUsd: admin.firestore.FieldValue.increment(amountUsd),
      })
      tx.update(db.collection('products').doc(productId), {
        invested: admin.firestore.FieldValue.increment(amountUsd),
        investorsCount: admin.firestore.FieldValue.increment(1),
      })
      tx.set(txRef, {
        userId: uid, type: 'investment', amountUsd,
        investmentId: investmentRef.id, productId,
        productName: productSnap.data()?.name,
        status: 'completed', createdAt: now,
      })
    })

    return { investmentId: investmentRef.id, txId: txRef.id }
  })
```

Export and deploy:
```bash
firebase deploy --only functions:createInvestment --project mombongo-dev
```

---

## Phase 4 — InvestModal + ProductDetailScreen wire (`mombongo-web`)

> Full spec in `S2-04-investment-flow.md`.

Create `src/components/InvestModal.tsx` — 3 steps: amount → summary → success.

Wire to `ProductDetailScreen.tsx` "Invest" button:
```typescript
const [investOpen, setInvestOpen] = useState(false)
// ...
<Button onClick={() => setInvestOpen(true)}>Investir</Button>
<InvestModal open={investOpen} onClose={() => setInvestOpen(false)} product={product} />
```

Create `src/services/investment.service.ts`:
```typescript
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'

export const investmentService = {
  createInvestment: (payload: { productId: string; amountUsd: number }) =>
    httpsCallable(functions, 'createInvestment')(payload),
}
```

---

## Phase 5 — Admin Product Management (`mombongo-admin`)

Create `src/pages/AdminProducts.tsx`:

**Features:**
- List all products from `products` Firestore collection
- "Nouveau produit" button → inline form or modal
- Toggle product `status` between `active` / `inactive`
- Delete draft products

**Create product form fields:**
```
name            text            "Pastèques de Songololo"
icon            emoji picker    🍉
category        select          légumes | fruits | grains | export | bourse
minInvest       number (USD)    min $10
duration        number (days)   e.g. 90
roi             number (%)      e.g. 18
location        text            "Songololo, Kongo Central"
description     textarea
stock           number          e.g. 500
unit            text            "tonnes"
```

**Admin service — `createProduct`:**
```typescript
// In admin.service.ts
import { collection, addDoc, serverTimestamp, getDocs, query, orderBy } from 'firebase/firestore'

async function createProduct(data: Omit<Product, 'id' | 'createdAt'>) {
  return addDoc(collection(db, 'products'), {
    ...data,
    status: 'active',
    invested: 0,
    investorsCount: 0,
    isActive: true,
    createdAt: serverTimestamp(),
  })
}

async function getProducts(): Promise<Product[]> {
  const snap = await getDocs(query(collection(db, 'products'), orderBy('createdAt', 'desc')))
  return snap.docs.map(d => ({ id: d.id, ...d.data() } as Product))
}
```

Add `AdminProducts` to the admin router and nav sidebar.

---

## Phase 6 — Admin Revenue Dashboard (`mombongo-admin`)

Wire `AdminDashboard.tsx` to real Firestore data (direct Firestore is allowed in `mombongo-admin`).

> Full spec in `S6-04-admin-kpi-dashboard.md`.

**Key KPI cards to show:**
| KPI | Source | Label |
|-----|--------|-------|
| Active users | `users` count | Utilisateurs actifs |
| Pending KYC | `users` where `kycStatus == 'pending'` | KYC en attente |
| Monthly investment volume | `transactions` this month | Volume mensuel ($) |
| Total deposits received | `deposits` where `status == 'completed'` | Dépôts totaux ($) |
| Active investments | `investments` where `status == 'active'` | Investissements actifs |
| Open products | `products` where `status == 'active'` | Produits actifs |

**Revenue sharing visibility:**
Add a `Revenu plateforme` KPI showing `monthlyVolumeUsd * 0.05` (5% platform fee — configurable).

Add activity feed: last 20 transactions via `onSnapshot` with type badge (deposit / investment / withdrawal).

---

## Prerequisites — seed one product for testing

Before Phase 4 E2E test, add one product to Firestore manually (or via `AdminProducts`):

```javascript
// Firebase Console → Firestore → products collection → Add document
{
  name: "Pastèques de Songololo",
  icon: "🍉",
  category: "légumes",
  minInvest: 50,
  duration: 90,
  roi: 18,
  location: "Songololo, Kongo Central",
  status: "active",
  isActive: true,
  invested: 0,
  investorsCount: 0,
  stock: 500,
  unit: "tonnes",
  createdAt: <timestamp>
}
```

---

## E2E Test Sequence (manual)

Once all phases deployed:

1. **Admin**: Go to `mombongo-admin` → AdminProducts → Create "Pastèques de Songololo" product
2. **Web**: Log in as investor at `mombongo-web.pages.dev`
3. **Web**: Go to Wallet → Dépôt → Airtel Money → enter PawaPay test phone → enter $200 → confirm
4. **PawaPay sandbox**: Simulate `COMPLETED` webhook (via PawaPay sandbox dashboard)
5. **Web**: Wallet balance shows $200 → Go to Market → click Pastèques → Investir $200
6. **Firestore Console**: Verify `investments` doc created, `walletUsd` decremented, `transactions` doc written
7. **Admin**: Go to AdminDashboard → KPIs show $200 volume, 1 active investment

---

## Definition of Done

- [ ] `initiateDeposit` accepted by PawaPay sandbox — returns `ACCEPTED`
- [ ] `pawapayWebhook` credits `walletUsd` on `COMPLETED` — verified via Firestore console
- [ ] `getDepositStatus` polling detects `completed` → UI shows "Dépôt réussi"
- [ ] `createInvestment` runs atomically — wallet decremented, investment + transaction docs created
- [ ] `InvestModal` shows success screen with `investmentId`
- [ ] Admin can create a product in `AdminProducts.tsx` — appears in web marketplace
- [ ] Admin KPI dashboard shows real volume, deposit, and investment counts
- [ ] Activity feed shows last 20 transactions in real time
- [ ] `npm run build` exits 0 (web + functions + admin)
- [ ] `npx vitest run` — all tests pass (web)

## Files to create / modify

| File | Repo | Change |
|------|------|--------|
| `src/payments/initiateDeposit.ts` | mombongo-functions | New |
| `src/payments/getDepositStatus.ts` | mombongo-functions | New |
| `src/payments/pawapayWebhook.ts` | mombongo-functions | New |
| `src/payments/initiateWithdraw.ts` | mombongo-functions | New (phase 1b) |
| `src/investments/createInvestment.ts` | mombongo-functions | New |
| `src/index.ts` | mombongo-functions | Export all above |
| `src/components/wallet/WalletModals.tsx` | mombongo-web | Wire DepositModal to initiateDeposit + polling |
| `src/lib/constants.ts` | mombongo-web | MOBILE_OPERATORS mapping |
| `src/services/investment.service.ts` | mombongo-web | New |
| `src/components/InvestModal.tsx` | mombongo-web | New |
| `src/pages/ProductDetailScreen.tsx` | mombongo-web | Wire "Invest" button to InvestModal |
| `src/pages/AdminProducts.tsx` | mombongo-admin | New — product CRUD |
| `src/services/admin.service.ts` | mombongo-admin | Add createProduct, getProducts |
| `src/pages/Admin.tsx` | mombongo-admin | Add AdminProducts to nav |
| `src/pages/AdminDashboard.tsx` | mombongo-admin | Wire to real Firestore KPIs |

## Branch and commit plan

```bash
# mombongo-functions
git checkout -b feature/sp-04-pawapay-e2e
# implement phases 1 + 3
npm run build
firebase deploy --only functions --project mombongo-dev
git commit -m "feat(sp-04): PawaPay deposit/webhook + createInvestment Cloud Functions"
git push origin feature/sp-04-pawapay-e2e

# mombongo-web
git checkout -b feature/sp-04-pawapay-e2e
# implement phases 2 + 4
npm run test:unit
npm run build
git commit -m "feat(sp-04): wire DepositModal to PawaPay + InvestModal + investment service"
git push origin feature/sp-04-pawapay-e2e

# mombongo-admin
git checkout -b feature/sp-04-pawapay-e2e
# implement phases 5 + 6
npm run build
git commit -m "feat(sp-04): AdminProducts CRUD + real-time KPI dashboard"
git push origin feature/sp-04-pawapay-e2e
```
