# S6-01 — Payments — PawaPay Mobile Money + Stripe Card Flow

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S6-01 |
| Sprint | Sprint 6 — Payments & Notifications |
| Branch | `feature/s6-01-payments` |
| Merges into | `dev` |
| Estimate | 3.5h |
| Dependencies | S2-04, S3-03 (walletUsd + walletCdf established) |

## Payment providers
| Method | Provider | Use case |
|--------|----------|----------|
| Mobile Money (Airtel, Orange, M-Pesa) | **PawaPay** | Primary payment method for DRC users |
| Visa / Mastercard | **Stripe** | International investors and diaspora |

## Status: Wallet UI complete via SP-02 — real payment backends remain

`src/components/wallet/WalletModals.tsx` was built in **SP-02** with two purpose-built modals:

**DepositModal** — operator picker + phone number field → maps directly to PawaPay deposit initiation.
**WithdrawModal** — operator picker + phone number → maps directly to PawaPay payout.

The Stripe card form (Visa/Mastercard tab) still needs to be added to `DepositModal`.

**What's still mock:** both modals use `setTimeout` fake-submit. No API is called, no Firestore doc written.

---

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | 🔨 Active | `initiateDeposit` · `pawapayWebhook` · `createStripePaymentIntent` · `stripeWebhook` · `initiateWithdraw` |
| `mombongo-web` | 🔨 Active | Replace mock submit in `DepositModal` / `WithdrawModal`; add Stripe card tab |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### Environment variables

```
# PawaPay
PAWAPAY_API_KEY=your_pawapay_api_key
PAWAPAY_ENVIRONMENT=sandbox   # or production
PAWAPAY_WEBHOOK_SECRET=your_webhook_secret

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

Set via Firebase CLI:
```bash
firebase functions:secrets:set PAWAPAY_API_KEY
firebase functions:secrets:set PAWAPAY_WEBHOOK_SECRET
firebase functions:secrets:set STRIPE_SECRET_KEY
firebase functions:secrets:set STRIPE_WEBHOOK_SECRET
```

---

### Part A — PawaPay (Mobile Money)

PawaPay REST API docs: https://docs.pawapay.io

#### initiateDeposit onCall

Create `src/payments/initiateDeposit.ts`:

```typescript
import axios from 'axios'
import { v4 as uuid } from 'uuid'

const PAWAPAY_BASE = process.env.PAWAPAY_ENVIRONMENT === 'production'
  ? 'https://api.pawapay.io'
  : 'https://api.sandbox.pawapay.io'

export const initiateDeposit = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { method, amountUsd, phone, operator, currency } = data

  if (amountUsd < 5) throw new functions.https.HttpsError('invalid-argument', 'Minimum deposit $5')

  const depositId = uuid()

  await db.collection('deposits').doc(depositId).set({
    userId: uid, depositId, method, amountUsd, currency,
    status: 'pending',
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  })

  if (method === 'mobile_money') {
    if (!phone || !operator)
      throw new functions.https.HttpsError('invalid-argument', 'phone and operator required')

    const response = await axios.post(
      `${PAWAPAY_BASE}/deposits`,
      {
        depositId,
        amount: String(amountUsd),
        currency,
        correspondent: operator,   // e.g. 'AIRTEL_DRC'
        payer: { type: 'MSISDN', address: { value: phone } },
        statementDescription: 'Dépôt Mombongo',
      },
      { headers: { Authorization: `Bearer ${process.env.PAWAPAY_API_KEY}` } }
    )

    if (response.data.status !== 'ACCEPTED') {
      await db.collection('deposits').doc(depositId).update({ status: 'failed' })
      throw new functions.https.HttpsError(
        'internal',
        `PawaPay rejected: ${response.data.rejectionReason?.rejectionCode ?? 'unknown'}`
      )
    }

    return { depositId, status: 'ACCEPTED', method: 'mobile_money' }
  }

  // Card path — return depositId; client calls createStripePaymentIntent next
  return { depositId, method: 'card' }
})
```

#### pawapayWebhook HTTP function

Create `src/payments/pawapayWebhook.ts`:

```typescript
import crypto from 'crypto'

export const pawapayWebhook = functions.https.onRequest(async (req, res) => {
  const signature = req.headers['x-pawapay-signature'] as string
  const expected  = crypto
    .createHmac('sha256', process.env.PAWAPAY_WEBHOOK_SECRET!)
    .update(JSON.stringify(req.body))
    .digest('hex')

  if (signature !== expected) { res.status(401).send('Invalid signature'); return }

  const { depositId, status } = req.body

  if (status !== 'COMPLETED') {
    await db.collection('deposits').doc(depositId).update({ status: 'failed' })
    res.status(200).send('OK'); return
  }

  const depositRef  = db.collection('deposits').doc(depositId)
  const depositSnap = await depositRef.get()
  if (!depositSnap.exists || depositSnap.data()?.status !== 'pending') {
    res.status(200).send('Already processed'); return
  }

  const { userId, amountUsd, currency } = depositSnap.data()!
  const walletField = currency === 'CDF' ? 'walletCdf' : 'walletUsd'
  const now         = admin.firestore.FieldValue.serverTimestamp()

  await db.runTransaction(async tx => {
    tx.update(db.collection('users').doc(userId), {
      [walletField]: admin.firestore.FieldValue.increment(amountUsd),
    })
    tx.update(depositRef, { status: 'completed', completedAt: now })
    tx.set(db.collection('transactions').doc(), {
      userId, type: 'deposit', method: 'mobile_money',
      amountUsd, currency, status: 'completed',
      pawapayDepositId: depositId, createdAt: now,
    })
  })

  res.status(200).send('OK')
})
```

---

### Part B — Stripe (Visa / Mastercard)

#### createStripePaymentIntent onCall

Create `src/payments/createStripePaymentIntent.ts`:

```typescript
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2023-10-16' })

export const createStripePaymentIntent = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { amountUsd, depositId }: { amountUsd: number; depositId: string } = data

  const paymentIntent = await stripe.paymentIntents.create({
    amount: Math.round(amountUsd * 100),
    currency: 'usd',
    metadata: { uid, depositId },
    automatic_payment_methods: { enabled: true },
  })

  return { clientSecret: paymentIntent.client_secret }
})
```

#### stripeWebhook HTTP function

Create `src/payments/stripeWebhook.ts`:

```typescript
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2023-10-16' })

export const stripeWebhook = functions.https.onRequest(async (req, res) => {
  const sig = req.headers['stripe-signature'] as string
  let event: Stripe.Event

  try {
    event = stripe.webhooks.constructEvent(req.rawBody, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch {
    res.status(400).send('Webhook signature verification failed'); return
  }

  if (event.type !== 'payment_intent.succeeded') { res.status(200).send('OK'); return }

  const pi        = event.data.object as Stripe.PaymentIntent
  const { uid, depositId } = pi.metadata
  const amountUsd = pi.amount / 100
  const now       = admin.firestore.FieldValue.serverTimestamp()

  const depositRef  = db.collection('deposits').doc(depositId)
  const depositSnap = await depositRef.get()
  if (!depositSnap.exists || depositSnap.data()?.status !== 'pending') {
    res.status(200).send('Already processed'); return
  }

  await db.runTransaction(async tx => {
    tx.update(db.collection('users').doc(uid), {
      walletUsd: admin.firestore.FieldValue.increment(amountUsd),
    })
    tx.update(depositRef, { status: 'completed', completedAt: now })
    tx.set(db.collection('transactions').doc(), {
      userId: uid, type: 'deposit', method: 'card',
      amountUsd, currency: 'USD', status: 'completed',
      stripePaymentIntentId: pi.id, createdAt: now,
    })
  })

  res.status(200).send('OK')
})
```

---

### Part C — PawaPay Payouts (Withdraw)

Create `src/payments/initiateWithdraw.ts`:

```typescript
export const initiateWithdraw = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { amountUsd, phone, operator, currency } = data

  const userSnap    = await db.collection('users').doc(uid).get()
  const walletField = currency === 'CDF' ? 'walletCdf' : 'walletUsd'
  const balance: number = userSnap.data()?.[walletField] ?? 0

  if (balance < amountUsd)
    throw new functions.https.HttpsError('failed-precondition', 'Insufficient balance')

  const payoutId = uuid()

  await db.runTransaction(async tx => {
    tx.update(db.collection('users').doc(uid), {
      [walletField]: admin.firestore.FieldValue.increment(-amountUsd),
    })
    tx.set(db.collection('withdrawals').doc(payoutId), {
      userId: uid, payoutId, amountUsd, currency,
      phone, operator, status: 'pending',
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    })
  })

  await axios.post(
    `${PAWAPAY_BASE}/payouts`,
    {
      payoutId,
      amount: String(amountUsd),
      currency,
      correspondent: operator,
      recipient: { type: 'MSISDN', address: { value: phone } },
      statementDescription: 'Retrait Mombongo',
    },
    { headers: { Authorization: `Bearer ${process.env.PAWAPAY_API_KEY}` } }
  )

  return { payoutId, status: 'ACCEPTED' }
})
```

Add `pawapayPayoutWebhook` following the same pattern as `pawapayWebhook` but updating the `withdrawals` collection.

Export all 5 functions in `src/index.ts`.

---

## mombongo-web

### Operator mapping

```typescript
export const MOBILE_OPERATORS = [
  { id: 'AIRTEL_DRC',  label: 'Airtel Money',  color: 'red'    },
  { id: 'ORANGE_DRC',  label: 'Orange Money',  color: 'orange' },
  { id: 'MPESA_DRC',   label: 'M-Pesa',        color: 'green'  },
]
```

### Wire DepositModal — Mobile Money

Replace the `setTimeout` in the mobile money submit handler:

```typescript
async function handleMobileMoneyDeposit() {
  setLoading(true)
  try {
    await httpsCallable(functions, 'initiateDeposit')({
      method: 'mobile_money',
      amountUsd: amount,
      phone,
      operator,
      currency: 'USD',
    })
    setStep('waiting')   // show "STK push envoyé — confirmez sur votre téléphone"
  } catch (err: any) {
    toast.error(err.message)
  } finally {
    setLoading(false)
  }
}
```

Poll deposit status via `httpsCallable(functions, 'getDepositStatus')({ depositId })` inside a `useQuery` with `refetchInterval: 3000`. Stop polling when `status === 'completed'` (set `enabled: status !== 'completed'`). No `onSnapshot` — `db` is not accessible from the frontend.

### Wire DepositModal — Card (new tab)

Install dependencies:
```bash
npm install @stripe/stripe-js @stripe/react-stripe-js
```

Add `VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...` to `.env.local`.

Add a "Carte bancaire" tab to `DepositModal` using `<CardElement>` from `@stripe/react-stripe-js`:

```typescript
const stripe  = useStripe()
const elements = useElements()

async function handleCardDeposit() {
  setLoading(true)
  try {
    // 1. Create pending deposit + get depositId
    const { data: { depositId } } = await httpsCallable(functions, 'initiateDeposit')({
      method: 'card', amountUsd: amount, currency: 'USD',
    }) as any

    // 2. Create PaymentIntent
    const { data: { clientSecret } } = await httpsCallable(functions, 'createStripePaymentIntent')({
      amountUsd: amount, depositId,
    }) as any

    // 3. Confirm payment with card element
    const result = await stripe!.confirmCardPayment(clientSecret, {
      payment_method: { card: elements!.getElement(CardElement)! },
    })

    if (result.error) throw new Error(result.error.message)
    toast.success('Paiement confirmé !')
    onClose()
  } catch (err: any) {
    toast.error(err.message)
  } finally {
    setLoading(false)
  }
}
```

### Wire WithdrawModal — PawaPay

Replace `setTimeout` in `WithdrawModal`:

```typescript
async function handleWithdraw() {
  setLoading(true)
  try {
    await httpsCallable(functions, 'initiateWithdraw')({
      amountUsd: amount, phone, operator, currency: 'USD',
    })
    toast.success(`Retrait en cours · 2–5 minutes · nouveau solde $${(balance - amount).toFixed(2)}`)
    onClose()
  } catch (err: any) {
    toast.error(err.message)
  } finally {
    setLoading(false)
  }
}
```

### Environment variable (web)

```
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
VITE_PAWAPAY_ENV=sandbox
```

---

## ✅ Definition of Done
- [ ] `initiateDeposit` accepted by PawaPay sandbox — STK push fires on test phone
- [ ] `pawapayWebhook` credits wallet on `COMPLETED` status
- [ ] `createStripePaymentIntent` returns `clientSecret`; card payment completes in Stripe sandbox
- [ ] `stripeWebhook` credits `walletUsd` after `payment_intent.succeeded`
- [ ] `initiateWithdraw` debits wallet + PawaPay payout initiated
- [ ] `npm run build` exits 0 (web + functions)

```bash
firebase deploy --only functions:initiateDeposit,pawapayWebhook,createStripePaymentIntent,stripeWebhook,initiateWithdraw
git commit -m "feat(s6-01): payments — PawaPay mobile money + Stripe card deposit/withdraw"
git push origin feature/s6-01-payments
```
