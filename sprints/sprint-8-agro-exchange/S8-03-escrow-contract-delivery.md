# S8-03 — Agro Exchange — Escrow, Digital Contract & Delivery Confirmation

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S8-03 |
| Sprint | Sprint 8 — Agro Exchange |
| Branch | `feature/s8-03-escrow-contract-delivery` |
| Merges into | `dev` |
| Estimate | 6h |
| Dependencies | S8-02 (match + negotiation complete, status = 'agreed') |

## Context

Once a buyer and seller agree on a price, the platform must:
1. Auto-generate a digital contract
2. Both parties e-sign (simple checkbox acceptance)
3. Buyer funds the escrow (PawaPay mobile money or wallet)
4. Seller confirms shipment
5. Buyer confirms delivery → escrow releases funds to seller's wallet

This closes the transaction loop and makes the Bourse an actual marketplace, not just a price discovery board.

**Escrow model for MVP**: funds held in the platform's Firestore `escrow_accounts` collection. Actual disbursement to seller is a Firestore debit on platform account + credit to seller's wallet. True bank escrow can be wired in a later sprint when a banking partner is confirmed.

---

## mombongo-functions

### generateContract onCall

Create `src/bourse/generateContract.ts`:

```typescript
// Called after both parties agree (match.status === 'agreed')
// Creates a bourse_contracts doc and sets match.status = 'contracted'
// Params: { matchId: string }
// Returns: { contractId: string }
export const generateContract = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { matchId } = data as { matchId: string }

  return await db.runTransaction(async tx => {
    const matchRef = db.collection('bourse_matches').doc(matchId)
    const matchSnap = await tx.get(matchRef)
    if (!matchSnap.exists) throw new functions.https.HttpsError('not-found', 'Match introuvable')

    const match = matchSnap.data()!
    if (match.status !== 'agreed')
      throw new functions.https.HttpsError('failed-precondition', 'Les parties n\'ont pas encore conclu un accord')
    if (match.buyerId !== uid && match.sellerId !== uid)
      throw new functions.https.HttpsError('permission-denied', 'Non autorisé')

    const now = admin.firestore.FieldValue.serverTimestamp()
    const deliveryDeadline = new Date()
    deliveryDeadline.setDate(deliveryDeadline.getDate() + 14) // 14 days default

    const contractRef = db.collection('bourse_contracts').doc()
    tx.set(contractRef, {
      matchId,
      sellerId: match.sellerId,
      buyerId: match.buyerId,
      commodity: match.commodity,
      quantityKg: match.quantityKg,
      pricePerKgCdf: match.agreedPricePerKgCdf,
      totalCdf: match.totalCdf,
      deliveryLocation: 'À confirmer par les parties',
      paymentTerms: 'escrow',
      deliveryDeadline: admin.firestore.Timestamp.fromDate(deliveryDeadline),
      sellerSignedAt: null,
      buyerSignedAt: null,
      status: 'pending_signatures',
      createdAt: now,
    })

    tx.update(matchRef, { status: 'contracted', contractId: contractRef.id, updatedAt: now })

    return { contractId: contractRef.id }
  })
})
```

### signContract onCall

Create `src/bourse/signContract.ts`:

```typescript
// Either party "signs" (checkbox acceptance). When both have signed, status → 'active'.
// Params: { contractId: string }
export const signContract = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { contractId } = data as { contractId: string }

  return await db.runTransaction(async tx => {
    const contractRef = db.collection('bourse_contracts').doc(contractId)
    const snap = await tx.get(contractRef)
    if (!snap.exists) throw new functions.https.HttpsError('not-found', 'Contrat introuvable')

    const contract = snap.data()!
    const isSeller = contract.sellerId === uid
    const isBuyer  = contract.buyerId  === uid
    if (!isSeller && !isBuyer) throw new functions.https.HttpsError('permission-denied', 'Non autorisé')

    const now = admin.firestore.FieldValue.serverTimestamp()
    const update: Record<string, any> = { updatedAt: now }

    if (isSeller && !contract.sellerSignedAt) update.sellerSignedAt = now
    if (isBuyer  && !contract.buyerSignedAt)  update.buyerSignedAt  = now

    // Check if both will have signed after this update
    const sellerSigned = isSeller ? true : !!contract.sellerSignedAt
    const buyerSigned  = isBuyer  ? true : !!contract.buyerSignedAt
    if (sellerSigned && buyerSigned) update.status = 'active'

    tx.update(contractRef, update)
    return { status: sellerSigned && buyerSigned ? 'active' : 'pending_signatures' }
  })
})
```

### fundEscrow onCall

Create `src/bourse/fundEscrow.ts`:

```typescript
// Buyer funds the escrow either from wallet (instant) or via PawaPay deposit (async).
// For wallet: deducts walletCdf from buyer, creates escrow_accounts doc with status = 'funded'.
// For mobile money: same PawaPay flow as initiateDeposit, but escrow is only marked 'funded'
//   after the depositWebhook fires and confirms the deposit.
// Params: { contractId: string; method: 'wallet' | 'mobile_money'; phone?: string; operatorId?: string }
export const fundEscrow = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { contractId, method, phone, operatorId } = data as {
    contractId: string; method: 'wallet' | 'mobile_money'; phone?: string; operatorId?: string
  }

  const contractSnap = await db.collection('bourse_contracts').doc(contractId).get()
  if (!contractSnap.exists) throw new functions.https.HttpsError('not-found', 'Contrat introuvable')

  const contract = contractSnap.data()!
  if (contract.buyerId !== uid) throw new functions.https.HttpsError('permission-denied', 'Seul l\'acheteur peut financer le séquestre')
  if (contract.status !== 'active') throw new functions.https.HttpsError('failed-precondition', 'Contrat non signé par les deux parties')

  const amountCdf = contract.totalCdf
  const now = admin.firestore.FieldValue.serverTimestamp()

  if (method === 'wallet') {
    return await db.runTransaction(async tx => {
      const userRef = db.collection('users').doc(uid)
      const userSnap = await tx.get(userRef)
      const walletCdf: number = userSnap.data()?.walletCdf ?? 0
      if (walletCdf < amountCdf)
        throw new functions.https.HttpsError('failed-precondition', 'Solde FC insuffisant')

      const escrowRef = db.collection('escrow_accounts').doc()
      tx.set(escrowRef, {
        contractId, matchId: contract.matchId,
        buyerId: uid, sellerId: contract.sellerId,
        amountCdf, depositedAt: now,
        status: 'funded',
        method: 'wallet',
        createdAt: now,
      })
      tx.update(userRef, { walletCdf: admin.firestore.FieldValue.increment(-amountCdf) })
      tx.update(db.collection('bourse_contracts').doc(contractId), {
        escrowId: escrowRef.id,
        escrowStatus: 'funded',
        updatedAt: now,
      })
      return { success: true, escrowStatus: 'funded' }
    })
  }

  // Mobile money: initiate PawaPay deposit (reuse existing initiateDeposit pattern)
  // The PawaPay webhook will call a new CF `escrowDepositWebhook` to mark escrow as funded
  // Implementation mirrors src/payments/initiateDeposit.ts
  // ... (similar axios call to PAWAPAY_BASE) ...
  throw new functions.https.HttpsError('unimplemented', 'Mobile money escrow — sprint suivant')
})
```

### confirmShipment + confirmDelivery + releaseEscrow

Create `src/bourse/confirmShipment.ts`:
```typescript
// Seller confirms they have shipped the goods.
// Updates bourse_matches.status = 'shipped', records shipmentDate.
// Params: { contractId: string; shipmentNote?: string }
```

Create `src/bourse/confirmDelivery.ts`:
```typescript
// Buyer confirms receipt of goods.
// Calls releaseEscrow internally → credits seller walletCdf.
// Params: { contractId: string }
```

Create `src/bourse/releaseEscrow.ts` (called internally by confirmDelivery, not exposed):
```typescript
// db.runTransaction:
//   escrow_accounts.status = 'released', releasedAt = now
//   seller.walletCdf += amountCdf
//   bourse_contracts.status = 'fulfilled'
//   bourse_matches.status = 'completed'
//   product_listings.status = 'sold' (update filledQuantityKg)
//   transactions doc: { type: 'bourse_sale', fromUid: buyerId, toUid: sellerId, amountCdf }
```

Export all in `src/index.ts`.

---

## mombongo-web

### Step 1 — ContractModal

New `src/components/ContractModal.tsx`:

```typescript
// Shown after a match reaches 'agreed' status.
// Steps:
//   1. Contract review (commodity, qty, price, delivery deadline, payment terms)
//   2. Signature step (checkbox "Je certifie avoir lu et j'accepte les termes du contrat")
//   3. After signing: buyer sees EscrowPaymentStep (wallet or mobile money)
//   4. Confirmation → success screen

interface ContractModalProps {
  open: boolean
  onClose: () => void
  contractId: string
  role: 'buyer' | 'seller'
  walletCdf: number
}
```

Key UI states:
- `pending_signatures`: show contract text + sign button
- `active` (both signed, buyer): show "Financer le séquestre" with wallet/mobile choice
- `active` (seller): show "En attente du paiement de l'acheteur"
- Buyer's escrow funded: show "En attente de l'expédition"
- Shipment confirmed: show "Confirmer la réception" (buyer) / "Livraison confirmée" (seller)
- Fulfilled: success screen with amount credited

### Step 2 — My Transactions page (Mes contrats)

Add a "Mes contrats" tab to `PortfolioScreen.tsx` (or a sub-route `/bourse/mes-contrats`):

```typescript
// Shows all bourse_contracts where user is buyer or seller.
// List view: commodity, qty, other party name, status chip, totalCdf.
// Click → open ContractModal for the ongoing contract.
```

### Step 3 — Wire into NegotiationModal (from S8-02)

After `acceptPrice` is called and match status = `'agreed'`, NegotiationModal shows:
```
"Accord trouvé ! Le prix de X FC/kg a été accepté."
[Générer le contrat →]  → calls generateContract CF → opens ContractModal
```

### Step 4 — i18n keys

```
bourse.contract.title      → "Contrat de vente" / "Sales contract"
bourse.contract.sign       → "Signer le contrat" / "Sign contract"
bourse.contract.fundEscrow → "Financer le séquestre" / "Fund escrow"
bourse.contract.escrowInfo → "Vos fonds seront libérés à la confirmation de livraison."
bourse.contract.confirmShip → "Confirmer l'expédition" / "Confirm shipment"
bourse.contract.confirmDel  → "Confirmer la réception" / "Confirm delivery"
bourse.contract.released    → "Fonds libérés — {amount} FC crédités" / "Funds released"
bourse.contract.dispute     → "Ouvrir un litige" / "Open a dispute"
```

---

## ✅ Definition of Done
- [ ] `generateContract` creates `bourse_contracts` doc
- [ ] `signContract` marks both-party signatures, status → `active` when both signed
- [ ] `fundEscrow` (wallet method) deducts `walletCdf` from buyer, creates `escrow_accounts`
- [ ] `confirmShipment` updates contract status → `shipped`
- [ ] `confirmDelivery` triggers `releaseEscrow` → credits seller `walletCdf`, status → `fulfilled`
- [ ] `ContractModal` covers all contract states end-to-end
- [ ] "Mes contrats" tab visible in portfolio/bourse area
- [ ] `npm run build` exits 0
- [ ] `npx vitest run` passes

```bash
firebase deploy --only functions:generateContract,functions:signContract,functions:fundEscrow,functions:confirmShipment,functions:confirmDelivery
git commit -m "feat(s8-03): escrow + digital contract + delivery confirmation"
```
