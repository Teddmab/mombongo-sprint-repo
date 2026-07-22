# SWS-03 — Triggered Notifications on Platform Events

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SWS-03 |
| Branch | `feature/sws-03-triggered-notifications` |
| Merges into | `dev` (mombongo-functions) |
| Estimate | 3h |
| Dependencies | SWS-01 (`notifyUser` helper), SWS-02 (users have phone + prefs stored) |

## Context

Wire `notifyUser` into the existing Firestore trigger CFs and the new Agro Exchange CFs so platform events automatically send WhatsApp/SMS to affected parties. No new CFs are exposed — this is purely internal wiring inside existing functions.

---

## mombongo-functions

### Events to wire

#### 1. Deposit confirmed (`onDepositCompleted` in `src/notifications/triggers.ts`)

After crediting the wallet, add:
```typescript
import { notifyUser } from './notifyUser'

// After existing FCM push:
await notifyUser(uid, `✅ Dépôt confirmé — ${formatCdf(amountCdf)} FC ont été crédités sur votre portefeuille Mombongo.`)
```

#### 2. Bourse investment created (`onBourseInvestmentCreated`)

```typescript
await notifyUser(uid, `📦 Votre investissement de ${formatCdf(amountCdf)} FC sur "${opportunityTitle}" est confirmé.`)
```

#### 3. Financing application status changed (`onFinancingStatusChanged`)

```typescript
const statusMsg: Record<string, string> = {
  approved:  `✅ Votre demande de financement pour ${cropType} a été approuvée.`,
  rejected:  `❌ Votre demande de financement pour ${cropType} n'a pas été retenue. Contactez un agent.`,
  funded:    `💸 Votre financement est décaissé. Fonds disponibles sur votre portefeuille.`,
}
const msg = statusMsg[newStatus]
if (msg) await notifyUser(applicantId, msg)
```

#### 4. Agro Exchange — match found (`createBuyerOrder` CF — S8-02)

After creating `bourse_matches`, notify both parties:
```typescript
for (const match of topMatches) {
  // Notify seller
  await notifyUser(
    match.data().sellerId,
    `🌿 Bonne nouvelle ! Un acheteur cherche ${quantityKg} kg de ${commodity} à ${maxPricePerKgCdf} FC/kg. Connectez-vous pour négocier.`
  )
}
// Notify buyer if matches found
if (candidates.length > 0) {
  await notifyUser(
    uid,
    `✅ ${candidates.length} offre(s) correspondent à votre demande de ${commodity}. Vérifiez vos correspondances.`
  )
}
```

#### 5. Contract signed by other party (`signContract` CF — S8-03)

```typescript
// After tx.update — if status becomes 'active':
if (sellerSigned && buyerSigned) {
  await notifyUser(contract.sellerId, `📝 Le contrat pour ${contract.commodity} est signé par les deux parties. En attente du paiement de l'acheteur.`)
  await notifyUser(contract.buyerId,  `📝 Contrat signé. Veuillez financer le séquestre pour débloquer la livraison.`)
}
```

#### 6. Escrow funded → notify seller (`fundEscrow` CF — S8-03)

```typescript
await notifyUser(
  contract.sellerId,
  `💰 L'acheteur a financé le séquestre (${formatCdf(amountCdf)} FC). Vous pouvez procéder à l'expédition.`
)
```

#### 7. Delivery confirmed → escrow released (`confirmDelivery` CF — S8-03)

```typescript
await notifyUser(
  contract.sellerId,
  `🎉 Livraison confirmée ! ${formatCdf(amountCdf)} FC ont été libérés sur votre portefeuille.`
)
await notifyUser(
  contract.buyerId,
  `✅ Réception confirmée. La transaction est terminée.`
)
```

#### 8. Harvest due reminder (`onHarvestDue` scheduled trigger)

```typescript
await notifyUser(
  farmer.uid,
  `🌾 Rappel : la récolte de ${farmer.cropType} est prévue dans 7 jours. Connectez-vous pour planifier votre mise en vente.`
)
```

---

### Shared formatter

Add to `src/lib/formatters.ts` (create if missing):

```typescript
export function formatCdf(amount: number): string {
  return new Intl.NumberFormat('fr-FR').format(Math.round(amount))
}
```

---

## ✅ Definition of Done
- [ ] `notifyUser` called in: `onDepositCompleted`, `onBourseInvestmentCreated`, `onFinancingStatusChanged`
- [ ] `notifyUser` called in: `createBuyerOrder` (match found), `signContract` (both signed), `fundEscrow`, `confirmDelivery`
- [ ] `notifyUser` called in: `onHarvestDue`
- [ ] `formatCdf` formatter in shared lib
- [ ] All wired CFs re-deployed
- [ ] Manual test: trigger a deposit in staging → WhatsApp/SMS received within 30s
- [ ] `npm run build` exits 0

```bash
firebase deploy --only \
  functions:onDepositCompleted \
  functions:onBourseInvestmentCreated \
  functions:onFinancingStatusChanged \
  functions:createBuyerOrder \
  functions:signContract \
  functions:fundEscrow \
  functions:confirmDelivery \
  functions:onHarvestDue
```
