# SU-02-04 — Bourse progress stepper (5 stages)

**Sprint:** SU-02 · Guided journey  
**Branch:** `feature/su-02-guided-journey`  
**Effort:** ~2 days

## Context
The Bourse screen shows listings as a flat list with no indication of where the farmer is in the selling process. This story adds a horizontal stepper at the top that shows the farmer their current position in the sale journey, making it clear what the next action is.

## The 5 stages

```
① Créer annonce  →  ② Trouver acheteur  →  ③ Confirmer commande  →  ④ Livrer  →  ⑤ Payé
```

State mapping from Bourse listing/transaction data:
- **No listing**: step 1 active → "Publiez votre première annonce"
- **Listing active, 0 orders**: step 2 active → "En attente d'acheteurs"
- **Order placed (pending)**: step 3 active → "Confirmez la commande"
- **Order confirmed, delivery pending**: step 4 active → "Organisez la livraison"
- **Transaction completed**: step 5 done → "Vente complétée — bon travail !"

## Implementation

### Logic
- Reads `useAgricultorBourseStatus()` — new selector on top of `useBourseFarmerListings` + `useTransactions`
- Returns `{ currentStep: 1|2|3|4|5, listing: Listing | null, order: Order | null }`
- `isDevMode()` returns step 2 (listing active, waiting for buyer)

### UI: `BourseProgressStepper` (`src/components/bourse/BourseProgressStepper.tsx`)
- Horizontal row of 5 steps with connecting lines
- Current step: filled accent circle + bold label
- Completed steps: checkmark + green line
- Inactive steps: gray circle + gray label
- Below stepper: one-line contextual hint ("Votre annonce de Maïs est visible par 120 acheteurs")
- Mobile: scrollable horizontally if viewport too narrow

### Placement
- Top of `AgricultorBourse` screen (mobile and desktop), below the header, above the listing list
- Shown only when the farmer has at least 1 active exploitation

## Acceptance criteria
- [ ] Stepper renders at the top of the Bourse screen for farmer role
- [ ] Step 1 highlighted when no listing exists; step 2+ require listing
- [ ] Clicking the active step's CTA navigates to the correct action
- [ ] Stepper does NOT render for merchant or investor roles
- [ ] Mobile: stepper scrolls horizontally without cutting off, no layout overflow
- [ ] `isDevMode()` renders stepper at step 2

## Smoke test steps
1. Log in as farmer with no listing → verify stepper shows step 1 highlighted
2. Create a listing → verify stepper advances to step 2 ("En attente d'acheteurs")
3. Simulate an order (set order status in Firestore) → verify step 3
4. Mobile viewport (375px) → verify stepper is readable, steps visible (scroll if needed)
5. Log in as merchant → verify stepper is NOT shown on their market screen
