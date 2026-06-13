# S2-04 — Marketplace — Investment Flow

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-04 |
| Sprint | Sprint 2 — Marketplace |
| Branch | `feature/s2-04-investment-flow` |
| Merges into | `dev` |
| Estimate | 4h |
| Dependencies | S2-03 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | InvestModal UI, call createInvestment function, optimistic update |
| `mombongo-functions` | 🔨 Active | createInvestment onCall — validates wallet, atomic Firestore transaction |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### Step 1 — createInvestment onCall function

Create `src/investments/createInvestment.ts`:

```typescript
import * as functions from 'firebase-functions'
import { db, admin } from '../lib/admin'

interface InvestPayload {
  productId: string
  amountUsd: number
}

export const createInvestment = functions.https.onCall(
  async (data: InvestPayload, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const { productId, amountUsd } = data
    if (!productId || amountUsd <= 0) {
      throw new functions.https.HttpsError('invalid-argument', 'Invalid payload')
    }

    const investmentRef = db.collection('investments').doc()
    const txRef = db.collection('transactions').doc()

    await db.runTransaction(async (tx) => {
      const userRef = db.collection('users').doc(uid)
      const productRef = db.collection('products').doc(productId)

      const [userSnap, productSnap] = await Promise.all([
        tx.get(userRef),
        tx.get(productRef),
      ])

      if (!userSnap.exists) {
        throw new functions.https.HttpsError('not-found', 'User not found')
      }
      if (!productSnap.exists || productSnap.data()?.status !== 'active') {
        throw new functions.https.HttpsError('not-found', 'Product not available')
      }

      const walletUsd: number = userSnap.data()?.walletUsd ?? 0
      const minInvest: number = productSnap.data()?.minInvest ?? 0

      if (amountUsd < minInvest) {
        throw new functions.https.HttpsError(
          'invalid-argument',
          `Minimum investment is $${minInvest}`
        )
      }
      if (walletUsd < amountUsd) {
        throw new functions.https.HttpsError('failed-precondition', 'Insufficient wallet balance')
      }

      const now = admin.firestore.FieldValue.serverTimestamp()
      const harvestDate = new Date()
      harvestDate.setDate(harvestDate.getDate() + (productSnap.data()?.duration ?? 30))

      // Create investment document
      tx.set(investmentRef, {
        investorId: uid,
        productId,
        amountUsd,
        roi: productSnap.data()?.roi,
        status: 'active',
        paymentStatus: 'completed',
        harvestDate: admin.firestore.Timestamp.fromDate(harvestDate),
        investedAt: now,
        productName: productSnap.data()?.name,
        productIcon: productSnap.data()?.icon,
      })

      // Deduct wallet
      tx.update(userRef, {
        walletUsd: admin.firestore.FieldValue.increment(-amountUsd),
        totalInvestedUsd: admin.firestore.FieldValue.increment(amountUsd),
      })

      // Update product progress
      tx.update(productRef, {
        invested: admin.firestore.FieldValue.increment(amountUsd),
        investorsCount: admin.firestore.FieldValue.increment(1),
      })

      // Create transaction record (write-only from server)
      tx.set(txRef, {
        userId: uid,
        type: 'investment',
        amountUsd,
        investmentId: investmentRef.id,
        productId,
        productName: productSnap.data()?.name,
        status: 'completed',
        createdAt: now,
      })
    })

    return { investmentId: investmentRef.id, txId: txRef.id }
  }
)
```

Export in `src/index.ts`:
```typescript
export { createInvestment } from './investments/createInvestment'
```

Deploy:
```bash
npm run build && firebase deploy --only functions:createInvestment
```

---

## mombongo-web

### Step 1 — investmentService

Create `src/services/investment.service.ts`:
```typescript
import { getFunctions, httpsCallable } from 'firebase/functions'
import { app } from '@/lib/firebase'

const functions = getFunctions(app, 'europe-west1')

export interface InvestPayload { productId: string; amountUsd: number }
export interface InvestResult  { investmentId: string; txId: string }

export const investmentService = {
  createInvestment: (payload: InvestPayload) =>
    httpsCallable<InvestPayload, InvestResult>(functions, 'createInvestment')(payload),
}
```

### Step 2 — InvestModal component

Create `src/components/InvestModal.tsx`:

```typescript
interface InvestModalProps {
  open: boolean
  onClose: () => void
  product: Pick<Product, 'id' | 'name' | 'icon' | 'roi' | 'minInvest' | 'duration'>
}
```

**3-step flow:**
1. **Amount step** — number input (min = `product.minInvest`), wallet balance displayed below, estimated return computed as `amount * roi / 100`
2. **Summary step** — read-only summary: product, amount, estimated return, harvest date, "Funds deducted from wallet"
3. **Success step** — confirmation with investment ID, link to portfolio

State:
```typescript
const [step, setStep] = useState<'amount' | 'summary' | 'success'>('amount')
const [amount, setAmount] = useState(product.minInvest)
const [loading, setLoading] = useState(false)
const [error, setError] = useState<string | null>(null)
const { userProfile, user } = useAuth()
```

Confirm handler:
```typescript
async function handleConfirm() {
  setLoading(true)
  setError(null)
  try {
    await investmentService.createInvestment({ productId: product.id, amountUsd: amount })
    queryClient.invalidateQueries({ queryKey: ['investments'] })
    queryClient.invalidateQueries({ queryKey: ['product', product.id] })
    setStep('success')
  } catch (e: any) {
    setError(e.message ?? t('invest.error'))
  } finally {
    setLoading(false)
  }
}
```

Key testids: `invest-amount-input`, `invest-next-btn`, `invest-confirm-btn`, `invest-success`.

### Step 3 — i18n keys

```
invest.title        → "Investir dans {{name}}" / "Invest in {{name}}"
invest.walletBal    → "Solde portefeuille" / "Wallet balance"
invest.estReturn    → "Rendement estimé" / "Estimated return"
invest.harvestDate  → "Date de récolte estimée" / "Estimated harvest"
invest.confirm      → "Confirmer l'investissement" / "Confirm investment"
invest.success      → "Investissement confirmé !" / "Investment confirmed!"
invest.error        → "Échec de l'investissement." / "Investment failed."
invest.insufficient → "Solde insuffisant" / "Insufficient balance"
```

---

## Unit Tests

`src/services/__tests__/investment.service.test.ts`:
```typescript
vi.mock('firebase/functions', () => ({
  getFunctions: vi.fn(),
  httpsCallable: vi.fn(() => vi.fn().mockResolvedValue({ data: { investmentId: 'inv-1' } })),
}))

it('calls createInvestment function with correct payload', async () => {
  const result = await investmentService.createInvestment({ productId: 'p1', amountUsd: 200 })
  expect(result.data.investmentId).toBe('inv-1')
})
```

`src/components/__tests__/InvestModal.test.tsx`:
```typescript
it('disables confirm when amount < minInvest', () => { ... })
it('shows success step after successful investment', async () => { ... })
it('shows error message on failure', async () => { ... })
```

---

## ✅ Definition of Done
- [ ] Investing $200 in Pastèques creates an `investments` doc in Firestore
- [ ] User `walletUsd` decreases by invested amount
- [ ] Product `invested` and `investorsCount` fields updated
- [ ] A `transactions` doc created with `status: completed`
- [ ] Error shown when wallet insufficient
- [ ] Modal shows confirmation with investmentId
- [ ] `npm run test:unit` passes (service + modal tests)
- [ ] `npm run build` exits 0 (web)
- [ ] `npm run build` exits 0 (functions)

```bash
# functions
firebase deploy --only functions:createInvestment

# web
git commit -m "feat(s2-04): investment flow — InvestModal + createInvestment function"
git push origin feature/s2-04-investment-flow
```
