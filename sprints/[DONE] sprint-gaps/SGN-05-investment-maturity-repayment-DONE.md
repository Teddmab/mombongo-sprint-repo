# SGN-05 — Investment Maturity & Repayment Tracking

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-05 |
| Repos | `mombongo-web`, `mombongo-functions`, `mombongo-admin` |
| Branch | `feature/sgn-05-investment-maturity` |
| Merges into | `dev` |
| Priority | P1 — Core investor trust feature (without this, investors can't see if/when they'll be paid) |
| Estimate | 6h |

## Why this matters
The investment flow (deposit → invest) is done. But what happens next? Investors currently have
no visibility into when their investment matures, what returns to expect, or how repayment works.
For non-tech users, "I invested $200 — what now?" with no feedback is a trust-killer.
This sprint closes the investment lifecycle loop.

---

## Data model
Existing `investments/{id}` doc:
```typescript
{
  investorId: string
  productId: string
  amountUsd: number
  investedAt: Timestamp
  status: "active" | "matured" | "repaid" | "cancelled"
  // NEW fields to add:
  maturityDate: Timestamp      // investedAt + product.durationDays
  expectedReturnUsd: number    // amountUsd * (1 + product.roi/100)
  repaidAt?: Timestamp
  repaidAmountUsd?: number
}
```

---

## Work items

### 1. Maturity date calculation on investment creation
Update `createInvestment` CF to compute and store `maturityDate` and `expectedReturnUsd`:
```typescript
// mombongo-functions/src/investments/createInvestment.ts
const product = (await db.collection('products').doc(productId).get()).data()
const maturityDate = new Date(now.toMillis() + product.durationDays * 86400 * 1000)
// Store maturityDate, expectedReturnUsd = amountUsd * (1 + product.roi / 100)
```

### 2. Investment status display on Portfolio screen (web)
**File**: `src/pages/PortfolioScreen.tsx` (or wherever investor portfolio lives)

Each investment card shows:
- Product name + crop type icon
- Amount invested: $200
- Expected return: $244 (+22%)
- Status pill: En cours / Arrivée à terme / Remboursé
- Maturity date: "Échéance: 15 sept. 2026" or countdown "Dans 23 jours"
- Progress bar (days elapsed / total duration)

```tsx
// Status colors:
// "active" → blue pill "En cours"
// "matured" → amber pill "Arrivée à terme" (investment period ended, awaiting repayment)
// "repaid" → green pill "Remboursé ✓" with repaid amount shown
```

### 3. Maturity notification
When an investment reaches its maturity date, a Cloud Function trigger should:
- Update investment `status: "matured"`
- Send push notification: "🎉 Votre investissement est arrivé à terme ! Vous recevrez bientôt votre remboursement."
- Send email notification (simple Firebase Extension or via Functions)

**New CF**: `onInvestmentMatured` (Firestore trigger or scheduled function that runs daily)
```typescript
// Scheduled: runs every 24h, checks investments where maturityDate <= now and status == 'active'
// Updates status to 'matured'
// Sends FCM push to investorId
```

### 4. Admin: Mark investment as repaid
In `mombongo-admin`, on the investments table:
- "Marquer comme remboursé" button → calls `repayInvestment({ investmentId, repaidAmountUsd })` CF
- CF atomically: updates investment status to 'repaid', credits wallet, creates transaction record
- Investor receives push notification: "✅ Votre remboursement de $244 a été crédité sur votre portefeuille."

**New CF**: `repayInvestment({ investmentId, repaidAmountUsd })`
```typescript
// Admin-only (check context.auth.token.admin or role)
// Atomically: investments/{id}.status = 'repaid', users/{investorId}.walletUsd += repaidAmountUsd
// Creates transactions doc: { type: 'repayment', investmentId, amountUsd, userId: investorId }
```

### 5. Investor Home — "Next repayment" widget
On InvestorHome, below the portfolio chart:
- "Prochain remboursement: 15 sept. 2026 — $244 attendus"
- If no upcoming repayment: "Aucun remboursement prévu ce mois-ci"

---

## Cloud Functions
- Update `createInvestment`: add `maturityDate`, `expectedReturnUsd` fields
- New `onInvestmentMatured`: scheduled daily check → update status + push notification
- New `repayInvestment({ investmentId, repaidAmountUsd })`: admin-triggered repayment

---

## Acceptance criteria
- [ ] New investments store `maturityDate` and `expectedReturnUsd`
- [ ] Portfolio shows maturity date, expected return, progress bar, and status pill for each investment
- [ ] Daily CF marks investments as "matured" when due
- [ ] Admin can mark investment as repaid — wallet credited
- [ ] Investor receives push notification on maturity and repayment
- [ ] `npx tsc --noEmit` passes across all repos

---

## Implementation Status
NOT DONE
