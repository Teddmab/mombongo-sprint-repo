# S6-01 — Payments — CinetPay Wallet Deposit Flow

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S6-01 |
| Sprint | Sprint 6 — Payments & Notifications |
| Branch | `feature/s6-01-cinetpay-deposit` |
| Merges into | `dev` |
| Estimate | 2h (UI done — backend wiring only) |
| Dependencies | S2-04, S3-03 (walletUsd + walletCdf established) |

## Status: Wallet UI complete via SP-02 — CinetPay backend remains

`src/components/wallet/WalletModals.tsx` was built in **SP-02** (439 lines) with two purpose-built modals:

**DepositModal** (green theme):
- Quick amounts ($50/$100/$250/$500) + custom input
- Mobile Money operator selection + phone number
- "Vous envoyez → reçu sur votre wallet" directional UX
- 800 ms fake-submit → `toast.success` with new balance
- Live balance updates in `ProfileScreen` via `useState(initialBalance)`

**WithdrawModal** (blue theme — visually distinct from Deposit):
- Before/after balance shown from operator step
- Max amount guard (remaining balance < $100 warning)
- "Vous retirez → envoyé vers Mobile Money" directional UX
- Same fake-submit pattern

**What's still mock:** both modals simulate submission with `setTimeout`. No Cloud Function is called. No Firestore doc is written. Balance update is local state only.

---

## Remaining work

### Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | 🔨 Active | `initiateCinetPayDeposit` onCall + `cinetpayWebhook` HTTP function |
| `mombongo-web` | 🔨 Active | Replace mock submit in `DepositModal` with real CinetPay redirect; read live balance from Firestore |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### Environment variables

Add to `.env` (functions):
```
CINETPAY_API_KEY=your_api_key
CINETPAY_SITE_ID=your_site_id
CINETPAY_SECRET_KEY=your_secret_key
CINETPAY_NOTIFY_URL=https://us-central1-YOUR_PROJECT.cloudfunctions.net/cinetpayWebhook
```

### initiateCinetPayDeposit onCall

Create `src/payments/initiateCinetPayDeposit.ts`:

```typescript
import axios from 'axios'

const CINETPAY_BASE = 'https://api-checkout.cinetpay.com/v2'

export const initiateCinetPayDeposit = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { amountUsd, currency = 'USD' }: { amountUsd: number; currency: 'USD' | 'CDF' } = data

  if (amountUsd < 5)
    throw new functions.https.HttpsError('invalid-argument', 'Minimum deposit $5')

  const transactionId = `DEP-${uid.slice(0, 6)}-${Date.now()}`

  await db.collection('deposits').doc(transactionId).set({
    userId: uid,
    transactionId,
    amountUsd,
    currency,
    status: 'pending',
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  })

  const response = await axios.post(`${CINETPAY_BASE}/payment`, {
    apikey:         process.env.CINETPAY_API_KEY,
    site_id:        process.env.CINETPAY_SITE_ID,
    transaction_id: transactionId,
    amount:         Math.round(amountUsd * 100),
    currency,
    description:    `Recharge portefeuille Mombongo — ${uid.slice(0, 6)}`,
    notify_url:     process.env.CINETPAY_NOTIFY_URL,
    return_url:     'https://app.mombongo.com/profile?deposit=success',
    cancel_url:     'https://app.mombongo.com/profile?deposit=cancelled',
    customer_id:    uid,
    channels:       'ALL',
  })

  if (response.data.code !== '201')
    throw new functions.https.HttpsError('internal', 'CinetPay checkout failed')

  return { paymentUrl: response.data.data.payment_url, transactionId }
})
```

### cinetpayWebhook HTTP function

Create `src/payments/cinetpayWebhook.ts`:

```typescript
export const cinetpayWebhook = functions.https.onRequest(async (req, res) => {
  const { cpm_trans_id, cpm_site_id, cpm_result } = req.body

  if (cpm_site_id !== process.env.CINETPAY_SITE_ID) { res.status(403).send('Invalid site_id'); return }
  if (cpm_result !== '00') {
    await db.collection('deposits').doc(cpm_trans_id).update({ status: 'failed' })
    res.status(200).send('OK'); return
  }

  const depositRef  = db.collection('deposits').doc(cpm_trans_id)
  const depositSnap = await depositRef.get()
  if (!depositSnap.exists || depositSnap.data()?.status !== 'pending') { res.status(200).send('Already processed'); return }

  const { userId, amountUsd, currency } = depositSnap.data()!

  await db.runTransaction(async tx => {
    const userRef     = db.collection('users').doc(userId)
    const walletField = currency === 'CDF' ? 'walletCdf' : 'walletUsd'
    const now         = admin.firestore.FieldValue.serverTimestamp()
    tx.update(userRef,    { [walletField]: admin.firestore.FieldValue.increment(amountUsd) })
    tx.update(depositRef, { status: 'completed', completedAt: now })
    tx.set(db.collection('transactions').doc(), {
      userId, type: 'deposit', amountUsd, currency, status: 'completed',
      cinetpayTransId: cpm_trans_id, createdAt: now,
    })
  })

  res.status(200).send('OK')
})
```

Export both in `src/index.ts`.

---

## mombongo-web

### Replace mock submit in DepositModal

In `src/components/wallet/WalletModals.tsx`, replace the `setTimeout` mock in `DepositModal`:

```typescript
async function handleSubmit() {
  setLoading(true)
  try {
    const result = await httpsCallable(functions, 'initiateCinetPayDeposit')({
      amountUsd: amount,
      currency: 'USD',
    }) as { data: { paymentUrl: string } }
    window.location.href = result.data.paymentUrl
  } catch (err: any) {
    toast.error(err.message)
    setLoading(false)
  }
}
```

### Detect return from CinetPay in ProfileScreen

```typescript
const [searchParams] = useSearchParams()
useEffect(() => {
  if (searchParams.get('deposit') === 'success') {
    toast.success(t('deposit.success'))
    queryClient.invalidateQueries({ queryKey: ['userProfile'] })
  }
}, [])
```

### Read live balance from Firestore

Replace the `useState(initialBalance)` pattern in ProfileScreen with a `useUserProfile()` hook reading `walletUsd` from the `users` Firestore doc.

---

## ✅ Definition of Done
- [ ] `initiateCinetPayDeposit` returns CinetPay payment URL
- [ ] `cinetpayWebhook` credits `walletUsd` / `walletCdf` on successful payment
- [ ] Deposit record created in `deposits` collection
- [ ] Return URL to `/profile?deposit=success` shows success toast + refreshes balance
- [ ] WithdrawModal wired to an equivalent `initiateWithdraw` function
- [ ] `npm run build` exits 0

```bash
firebase deploy --only functions:initiateCinetPayDeposit,cinetpayWebhook
git commit -m "feat(s6-01): CinetPay deposit — replace mock in WalletModals + initiate + webhook"
git push origin feature/s6-01-cinetpay-deposit
```
