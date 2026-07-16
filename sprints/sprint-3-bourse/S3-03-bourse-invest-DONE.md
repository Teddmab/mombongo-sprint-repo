# S3-03 — Bourse — Invest in Transport Route

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S3-03 |
| Sprint | Sprint 3 — Bourse |
| Branch | `feature/s3-03-bourse-invest` |
| Merges into | `dev` |
| Estimate | 3h |
| Dependencies | S3-02, S2-04 (investment flow pattern established) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | BourseInvestModal — CDF amount input, confirm, success |
| `mombongo-functions` | 🔨 Active | createBourseInvestment onCall |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### createBourseInvestment onCall

Create `src/bourse/createBourseInvestment.ts` — same pattern as `createInvestment` (S2-04) but:
- Collection: `bourse_investments` (uses `investorId` + `amountCdf`)
- Wallet field: `walletCdf` (CDF balance, separate from `walletUsd`)
- Updates `bourse_opportunities.filledKg` by `amountCdf / pricePerKg`
- Minimum: `amountCdf >= 10000` (enforced in security rules)

```typescript
export const createBourseInvestment = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { opportunityId, amountCdf }: { opportunityId: string; amountCdf: number } = data
  if (amountCdf < 10_000) throw new functions.https.HttpsError('invalid-argument', 'Minimum 10,000 FC')

  await db.runTransaction(async tx => {
    const userRef = db.collection('users').doc(uid)
    const oppRef  = db.collection('bourse_opportunities').doc(opportunityId)
    const [userSnap, oppSnap] = await Promise.all([tx.get(userRef), tx.get(oppRef)])

    if (!oppSnap.exists || oppSnap.data()?.status !== 'open')
      throw new functions.https.HttpsError('not-found', 'Opportunity not available')

    const walletCdf: number = userSnap.data()?.walletCdf ?? 0
    if (walletCdf < amountCdf)
      throw new functions.https.HttpsError('failed-precondition', 'Insufficient CDF balance')

    const now = admin.firestore.FieldValue.serverTimestamp()
    const invRef = db.collection('bourse_investments').doc()
    const txRef  = db.collection('transactions').doc()

    tx.set(invRef, {
      investorId: uid,
      opportunityId,
      amountCdf,
      roi: oppSnap.data()?.roi,
      status: 'active',
      route: oppSnap.data()?.route,
      commodity: oppSnap.data()?.commodity,
      investedAt: now,
    })

    tx.update(userRef, { walletCdf: admin.firestore.FieldValue.increment(-amountCdf) })
    tx.update(oppRef, {
      filledKg: admin.firestore.FieldValue.increment(amountCdf / (oppSnap.data()?.targetCdf / oppSnap.data()?.capacityKg)),
      investorsCount: admin.firestore.FieldValue.increment(1),
    })

    tx.set(txRef, {
      userId: uid,
      type: 'bourse_investment',
      amountCdf,
      opportunityId,
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

### BourseInvestModal

Same 3-step flow as `InvestModal` (S2-04) but:
- Currency: **CDF** (Francs congolais), display with `formatCdf()`
- Input shows `walletCdf` balance from user profile
- Minimum is `opportunity.minInvestCdf` (10,000 FC default)
- Estimated return: `amountCdf * roi / 100` displayed in CDF

```typescript
// investmentService addition:
createBourseInvestment: (payload: { opportunityId: string; amountCdf: number }) =>
  httpsCallable(functions, 'createBourseInvestment')(payload),
```

Key testids: `bourse-invest-amount-input`, `bourse-invest-confirm-btn`, `bourse-invest-success`.

### i18n keys

```
bourse.invest.title    → "Investir dans ce convoi" / "Invest in this route"
bourse.invest.walletCdf → "Solde FC" / "CDF balance"
bourse.invest.estReturn → "Retour estimé" / "Estimated return"
bourse.invest.success   → "Investissement confirmé !" / "Investment confirmed!"
```

---

## ✅ Definition of Done
- [ ] CDF investment creates `bourse_investments` doc
- [ ] `walletCdf` deducted from user
- [ ] `filledKg` on opportunity updated
- [ ] Error shown for insufficient CDF balance
- [ ] `npm run test:unit` passes
- [ ] Both `npm run build` pass (web + functions)

```bash
firebase deploy --only functions:createBourseInvestment
git commit -m "feat(s3-03): bourse invest modal + createBourseInvestment function"
git push origin feature/s3-03-bourse-invest
```
