# SG-13 — Payment Card (Stripe) + Subscription Persistence

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-13 |
| Sprint | Sprint Gaps 13 |
| Branch | `feature/sg-13-payment-card-subscription` |
| Merges into | `dev` |
| Estimate | 6h |
| Dependencies | S6-01 (PaymentModal exists), S6-03/S6-04 (PawaPay integrated) |

---

## Context

Two payment stubs remain from Sprint 6:

1. **`PaymentModal.tsx` card tab**: Always shows a hard-coded error "Paiement par carte non disponible
   pour le moment" — no Stripe integration, just a locked-off dead end.

2. **`SubscriptionModal.tsx`**: Processes wallet payment successfully (PawaPay path works) but does
   NOT write the subscription tier to Firestore or update `userProfile.tier`. The subscription modal
   fires a toast "Abonnement activé" but the user stays on `tier: 'free'` forever.

---

## Part 1: Stripe Card Payment

### Architecture

Stripe integration follows the same architecture rule: **no direct Stripe SDK from frontend**. All
Stripe API calls go through Cloud Functions.

Flow:
```
Frontend → createStripePaymentIntent CF → Stripe API → PaymentIntent { clientSecret }
Frontend → Stripe.js (stripe.confirmCardPayment(clientSecret)) → Stripe handles 3DS
Stripe webhook → /stripe-webhook (separate HTTP function) → writes payment result to Firestore
Frontend → polls or listens for payment completion → updates wallet
```

### Cloud Function: `createStripePaymentIntent`
```typescript
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2024-12-18.acacia' })

// Params: { amountUsd: number; description: string }
// Returns: { clientSecret: string; paymentIntentId: string }
export const createStripePaymentIntent = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { amountUsd, description } = data
  if (!amountUsd || amountUsd < 1)
    throw new functions.https.HttpsError('invalid-argument', 'Montant invalide')

  const userSnap = await db.collection('users').doc(uid).get()
  const user = userSnap.data() ?? {}

  // Get or create Stripe customer
  let stripeCustomerId = user.stripeCustomerId
  if (!stripeCustomerId) {
    const customer = await stripe.customers.create({
      email: user.email,
      name: user.displayName ?? user.fullName,
      metadata: { userId: uid },
    })
    stripeCustomerId = customer.id
    await db.collection('users').doc(uid).update({ stripeCustomerId })
  }

  const intent = await stripe.paymentIntents.create({
    amount: Math.round(amountUsd * 100),  // Stripe uses cents
    currency: 'usd',
    customer: stripeCustomerId,
    description,
    metadata: { userId: uid },
  })

  return { clientSecret: intent.client_secret!, paymentIntentId: intent.id }
})
```

### Cloud Function: `confirmStripePayment` (webhook handler)
```typescript
// HTTP function (not onCall) — Stripe webhook endpoint
// Verifies signature, then on payment_intent.succeeded:
//   1. Updates wallet balance in Firestore (db.collection('users').doc(uid).update({ walletUsd: increment(amount) }))
//   2. Creates a transaction doc
//   3. Sends push notification
export const stripeWebhook = functions.region('europe-west1').https.onRequest(async (req, res) => {
  const sig = req.headers['stripe-signature']!
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(req.rawBody, sig, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch {
    res.status(400).send('Webhook signature invalid')
    return
  }

  if (event.type === 'payment_intent.succeeded') {
    const intent = event.data.object as Stripe.PaymentIntent
    const uid = intent.metadata.userId
    const amountUsd = intent.amount / 100

    await db.runTransaction(async tx => {
      const userRef = db.collection('users').doc(uid)
      tx.update(userRef, { walletUsd: FieldValue.increment(amountUsd) })
      const txRef = db.collection('transactions').doc()
      tx.set(txRef, {
        userId: uid,
        type: 'deposit',
        amountUsd,
        method: 'stripe_card',
        paymentIntentId: intent.id,
        status: 'completed',
        createdAt: FieldValue.serverTimestamp(),
      })
    })
  }

  res.json({ received: true })
})
```

### Frontend: `PaymentModal.tsx` card tab

Replace the locked error message with the real Stripe.js flow:

```tsx
import { loadStripe } from '@stripe/stripe-js'
import { CardElement, Elements, useStripe, useElements } from '@stripe/react-stripe-js'

const stripePromise = loadStripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY)

function CardPaymentForm({ amountUsd, description, onSuccess }: CardPaymentFormProps) {
  const stripe = useStripe()
  const elements = useElements()
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  async function handleSubmit() {
    if (!stripe || !elements) return
    setLoading(true); setError(null)
    try {
      const fn = httpsCallable(functions, 'createStripePaymentIntent')
      const { data } = await fn({ amountUsd, description }) as any
      const result = await stripe.confirmCardPayment(data.clientSecret, {
        payment_method: { card: elements.getElement(CardElement)! },
      })
      if (result.error) throw result.error
      onSuccess()
    } catch (e: any) {
      setError(e.message ?? 'Erreur de paiement')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="space-y-4">
      <div className="border border-gray-200 rounded-xl p-3">
        <CardElement />
      </div>
      {error && <p className="text-sm text-red-600">{error}</p>}
      <button onClick={handleSubmit} disabled={loading || !stripe}>
        {loading ? 'Traitement…' : `Payer ${amountUsd} USD`}
      </button>
    </div>
  )
}

// Wrap in <Elements stripe={stripePromise}> where the card tab renders
```

Add env var: `VITE_STRIPE_PUBLISHABLE_KEY` (public key, safe to expose in frontend).

---

## Part 2: Subscription Tier Persistence

### Cloud Function: `activateSubscription`
```typescript
// Params: { tier: 'starter' | 'growth' | 'pro'; paymentMethod: 'wallet' | 'stripe'; paymentIntentId? }
export const activateSubscription = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { tier, paymentMethod, paymentIntentId } = data
  const PRICES: Record<string, number> = { starter: 5, growth: 15, pro: 30 }
  const priceUsd = PRICES[tier]
  if (!priceUsd) throw new functions.https.HttpsError('invalid-argument', 'Tier invalide')

  await db.runTransaction(async tx => {
    const userRef = db.collection('users').doc(uid)
    const userSnap = await tx.get(userRef)
    const user = userSnap.data() ?? {}

    if (paymentMethod === 'wallet') {
      if ((user.walletUsd ?? 0) < priceUsd)
        throw new functions.https.HttpsError('failed-precondition', 'Solde insuffisant')
      tx.update(userRef, {
        walletUsd: FieldValue.increment(-priceUsd),
        tier,
        tierActivatedAt: FieldValue.serverTimestamp(),
        tierExpiresAt: new Date(Date.now() + 30 * 86400000),  // 30 days
      })
    } else {
      // Stripe: verify paymentIntentId succeeded before activating
      // (optionally verify via stripe.paymentIntents.retrieve — or trust webhook already updated wallet)
      tx.update(userRef, {
        tier,
        tierActivatedAt: FieldValue.serverTimestamp(),
        tierExpiresAt: new Date(Date.now() + 30 * 86400000),
        stripePaymentIntentId: paymentIntentId ?? null,
      })
    }

    const subRef = db.collection('subscriptions').doc()
    tx.set(subRef, {
      userId: uid,
      tier,
      priceUsd,
      paymentMethod,
      status: 'active',
      activatedAt: FieldValue.serverTimestamp(),
      expiresAt: new Date(Date.now() + 30 * 86400000),
    })
  })

  return { ok: true, tier }
})
```

### Frontend: `SubscriptionModal.tsx`

Replace the direct wallet deduction with `activateSubscription` CF call:
```typescript
// Old code (calls processWalletPayment directly):
await httpsCallable(functions, 'processWalletPayment')({ amountUsd: price, reason: 'subscription' })
toast.success("Abonnement activé")

// New code:
await httpsCallable(functions, 'activateSubscription')({ tier, paymentMethod: 'wallet' })
qc.invalidateQueries({ queryKey: ['userProfile'] })   // refresh tier in AuthContext
toast.success(`Abonnement ${tier} activé !`)
```

After invalidating `userProfile`, the tier badge throughout the app updates automatically.

---

## Environment Variables to Add

| Variable | Where | Description |
|----------|-------|-------------|
| `VITE_STRIPE_PUBLISHABLE_KEY` | mombongo-web `.env` + CI | Stripe public key (safe to expose) |
| `STRIPE_SECRET_KEY` | mombongo-functions env | Stripe secret key (server-only) |
| `STRIPE_WEBHOOK_SECRET` | mombongo-functions env | Webhook signing secret |

---

## Acceptance Criteria
- [ ] `createStripePaymentIntent` CF creates PaymentIntent and returns `clientSecret`
- [ ] Stripe webhook handler verifies signature + credits wallet on success
- [ ] `PaymentModal` card tab shows real `<CardElement>` (not locked error message)
- [ ] Card payment flow completes end-to-end (card input → confirm → wallet credit)
- [ ] `activateSubscription` CF deducts wallet and updates `users/{uid}.tier`
- [ ] `SubscriptionModal` calls `activateSubscription` (not `processWalletPayment`)
- [ ] `userProfile.tier` updates in UI after subscription (via `qc.invalidateQueries`)
- [ ] `subscriptions/{id}` doc created with 30-day expiry
- [ ] Dev mode: card payment shows "Mode dev — paiement simulé" and calls `onSuccess` directly
