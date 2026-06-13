# S4-02 — Financing — Farmer Detail and Fund Flow

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S4-02 |
| Sprint | Sprint 4 — Financing |
| Branch | `feature/s4-02-farmer-detail` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S4-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /financing/:id — farmer detail, tranche timeline, FundModal |
| `mombongo-functions` | 🔨 Active | createFinancingApplication onCall |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### createFinancingApplication onCall

Create `src/financing/createFinancingApplication.ts`:

```typescript
export const createFinancingApplication = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { farmerId, amountUsd }: { farmerId: string; amountUsd: number } = data
  if (amountUsd < 50)
    throw new functions.https.HttpsError('invalid-argument', 'Minimum $50')

  await db.runTransaction(async tx => {
    const userRef   = db.collection('users').doc(uid)
    const farmerRef = db.collection('farmers').doc(farmerId)
    const [userSnap, farmerSnap] = await Promise.all([tx.get(userRef), tx.get(farmerRef)])

    if (!farmerSnap.exists || !['approved', 'active'].includes(farmerSnap.data()?.status))
      throw new functions.https.HttpsError('not-found', 'Farmer not available for funding')

    const walletUsd: number = userSnap.data()?.walletUsd ?? 0
    if (walletUsd < amountUsd)
      throw new functions.https.HttpsError('failed-precondition', 'Insufficient USD balance')

    const remaining = farmerSnap.data()!.requestedAmountUsd - farmerSnap.data()!.disbursedAmountUsd
    if (amountUsd > remaining)
      throw new functions.https.HttpsError('invalid-argument', `Max fundable: $${remaining}`)

    const now   = admin.firestore.FieldValue.serverTimestamp()
    const appRef = db.collection('financing_applications').doc()
    const txRef  = db.collection('transactions').doc()

    tx.set(appRef, {
      farmerId,
      investorId: uid,
      amountUsd,
      tranches: [{ amountUsd, status: 'disbursed', disbursedAt: now }],
      status: 'active',
      cropType: farmerSnap.data()?.cropType,
      createdAt: now,
    })

    tx.update(userRef,   { walletUsd: admin.firestore.FieldValue.increment(-amountUsd) })
    tx.update(farmerRef, {
      disbursedAmountUsd: admin.firestore.FieldValue.increment(amountUsd),
      status: 'active',
    })

    tx.set(txRef, {
      userId: uid,
      type: 'financing',
      amountUsd,
      farmerId,
      status: 'completed',
      createdAt: now,
    })
  })

  return { success: true }
})
```

Export in `src/index.ts`.

---

## mombongo-web

### Step 1 — useFarmer hook

Add to `src/hooks/useFinancing.ts`:

```typescript
import { doc, getDoc } from 'firebase/firestore'

export function useFarmer(id: string) {
  return useQuery({
    queryKey: ['farmer', id],
    queryFn: async () => {
      if (isDevMode()) return (MOCK_FARMERS.find(f => f.id === id) ?? null) as Farmer | null
      const snap = await getDoc(doc(db, 'farmers', id))
      return snap.exists() ? ({ id: snap.id, ...snap.data() } as Farmer) : null
    },
    enabled: !!id,
  })
}

export function useMyFinancingApplications() {
  const { user } = useAuth()
  return useQuery({
    queryKey: ['my-financing', user?.uid],
    queryFn: async () => {
      if (!user?.uid) return []
      if (isDevMode()) return []
      const snap = await getDocs(
        query(
          collection(db, 'financing_applications'),
          where('investorId', '==', user.uid),
          orderBy('createdAt', 'desc')
        )
      )
      return snap.docs.map(d => ({ id: d.id, ...d.data() }))
    },
    enabled: !!user?.uid,
  })
}
```

Add `financingService` to `src/services/investmentService.ts`:
```typescript
createFinancingApplication: (payload: { farmerId: string; amountUsd: number }) =>
  httpsCallable(functions, 'createFinancingApplication')(payload),
```

### Step 2 — Farmer detail screen

Route: `/financing/:id`

```typescript
const { id } = useParams<{ id: string }>()
const { data: farmer, isLoading } = useFarmer(id!)
```

Sections:
1. **Header** — farmer photo/avatar, name, region + crop chip, `status` chip
2. **Farm stats row** — farm size (ha) / crop type / harvest date / requested amount
3. **Funding progress bar** — `disbursedAmountUsd / requestedAmountUsd`, percentage label
4. **Tranche timeline** — list from `financing_applications` where this farmer is funded (shows investor count, amounts)
5. **Description / about the farmer** — `farmer.description` (optional field)
6. **Sticky Fund button** — `data-testid="farmer-fund-cta"`, opens FundModal, disabled when `status === 'completed'`

### Step 3 — FundModal (3-step flow same as InvestModal)

```tsx
// Step 1 — Amount input
<p>{t('financing.walletBalance')}: {formatUsd(userProfile?.walletUsd ?? 0)}</p>
<input
  data-testid="farmer-fund-amount-input"
  type="number"
  min={50}
  max={farmer.requestedAmountUsd - farmer.disbursedAmountUsd}
  value={amount}
  onChange={e => setAmount(Number(e.target.value))}
/>
<p className="text-sm text-gray-500">
  {t('financing.remaining')}: {formatUsd(farmer.requestedAmountUsd - farmer.disbursedAmountUsd)}
</p>

// Step 2 — Confirm
<p>{t('financing.fundConfirm', { amount: formatUsd(amount), name: farmer.name })}</p>
<button data-testid="farmer-fund-confirm-btn" onClick={submit}>{t('financing.confirm')}</button>

// Step 3 — Success
<p data-testid="farmer-fund-success">{t('financing.success')}</p>
```

On submit: call `createFinancingApplication({ farmerId: id!, amountUsd: amount })`, invalidate `['farmer', id]` and `['my-financing']` queries.

### Step 4 — i18n keys

```
financing.walletBalance  → "Solde USD" / "USD balance"
financing.remaining      → "Restant à financer" / "Remaining to fund"
financing.fundConfirm    → "Financer {{name}} pour {{amount}}" / "Fund {{name}} for {{amount}}"
financing.confirm        → "Confirmer" / "Confirm"
financing.success        → "Financement confirmé !" / "Funding confirmed!"
financing.minAmount      → "Minimum $50" / "Minimum $50"
```

---

## ✅ Definition of Done
- [ ] `/financing/:id` loads farmer from Firestore
- [ ] Funding progress bar reflects real `disbursedAmountUsd / requestedAmountUsd`
- [ ] `createFinancingApplication` deducts `walletUsd` and updates farmer
- [ ] Error shown for insufficient USD balance
- [ ] `data-testid="farmer-fund-cta"` present
- [ ] `npm run build` exits 0

```bash
firebase deploy --only functions:createFinancingApplication
git commit -m "feat(s4-02): farmer detail + fund flow + createFinancingApplication function"
git push origin feature/s4-02-farmer-detail
```
