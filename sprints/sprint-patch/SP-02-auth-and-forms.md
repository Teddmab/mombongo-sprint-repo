# SP-02 — Auth Flow, Action Modals & Wallet

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-02 |
| Sprint | Patch — Point-in-time corrections & quick implementations |
| Branch | `feature/sp-02-auth-and-forms` |
| PR | [#24](https://github.com/Teddmab/mombongo-web/pull/24) |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Date | 2026-06-01 |
| Type | Quick implementation + correction |

## Scope — `mombongo-web`

Extends SP-01 with three areas:
1. Auth screen role selection UX
2. Action modals replacing all "bientôt disponible" toast stubs
3. /report/new role-based form screen (was a blank stub)
4. Wallet deposit/withdraw visual differentiation with live balance

---

## Changes

### 1. Auth Screen — 2-step role selection

**Problem:** Role selector was buried at the bottom of the login form, after credentials and Google button. Users had no context for what they were signing up as.

**Fix:** Added `step: "role" | "auth"` state.
- Step 1: 4 large role cards with themed Lucide icons + one-line description. "Continuer" button advances to step 2.
- Step 2: Login/signup form with selected role shown as a chip at the top. "Modifier" returns to step 1.
- Icons: TrendingUp (green) · Leaf (lime) · Store (purple) · ClipboardList (amber) — consistent with nav role badge colors.

**Tests updated:** `renderScreen()` in `AuthScreen.test.tsx` now clicks through the role step before interacting with form fields. All 24 tests still pass.

---

### 2. Action Modals (`src/components/forms/ActionForms.tsx`)

Replaced all "bientôt disponible" toast stubs with real modal forms:

| Modal | Role | Screen | Fields |
|-------|------|--------|--------|
| `PublierProduitModal` | Agriculteur | Market | Nom, catégorie, qty, prix/unit FC, région, date récolte |
| `CommanderModal` | Commerçant | Market | Qty, adresse, date, paiement, notes |
| `MettreEnVenteModal` | Agriculteur | Bourse | Culture, qty, prix, disponibilité |
| `ReserverLotModal` | Commerçant | Bourse | Parts (max = spotsLeft), paiement, lot details inline |
| `PreAcheterModal` | Commerçant | Financement | Culture, montant, livraison, conditions |
| `PublierPourAgriculteurModal` | Agent | Market | Farmer selector (from agentFarmers), product fields, region pre-filled |
| `PublierLotModal` | Commerçant | Bourse | Lot type picker (Transport/Stockage/Transformation), origin, volume, price, spots |

All modals: spring animation, 800ms fake-submit loading state, `toast.success` on completion.

---

### 3. /report/new — Role-based report screen

**Problem:** Screen was a 1-line stub (`AgentReportScreen = () => <div>AgentReportScreen</div>`).

**Fix:** Full role-dispatched implementation.

**Agent terrain — Rapport de visite terrain** (6 sections):
1. Farmer selector from assigned list, visit date, agent name (read-only)
2. Visual condition scale 1–5 (color-coded), growth stage, surface
3. Problems checkboxes (ravageurs, maladies, sécheresse, pluies, sol, aucun)
4. Financing: disbursed + additional needs
5. Recommendations textarea + next visit date
6. Photo placeholder upload

Desktop: sticky sidebar with live summary of entered values.

**Agriculteur — Signalement culture** (4 sections):
1. Crop selector + date + growth stage pill-picker
2. Condition scale — red alert when critical (≤ 2)
3. Urgency checkboxes (irrigation, traitement, visite, financement)
4. Message to agent + agent card

**Investor/Merchant:** Access-denied screen with role-specific explanation.

---

### 4. Wallet — DepositModal & WithdrawModal (`src/components/wallet/WalletModals.tsx`)

**Problem:** PaymentModal used for both deposit and withdraw looked identical — same green color scheme, same "paying" UX. Confusing direction of money flow.

**Fix:** Two purpose-built modals.

**DepositModal** (green):
- "Vous envoyez → reçu sur votre wallet"
- BEFORE balance shown only from confirm step
- Quick amounts ($50/$100/$250/$500) + custom input
- Mobile Money operator selection + phone number
- Success: "+$X ajoutés · nouveau solde"

**WithdrawModal** (blue — visually distinct):
- "Vous retirez → envoyé vers Mobile Money"
- BEFORE → AFTER balance visible from operator step
- Max amount = current balance (guard + error message)
- Low-balance warning when remaining < $100
- Success: "Retrait en cours · 2–5 minutes · nouveau solde"

**Live balance:** `walletBalance` tracked as `useState(initialBalance)` in both DesktopProfile and MobileProfile. Updates immediately on success (deposit +, withdraw -). Investor starts at $1,245.80, Merchant at $2,100.00.

---

## Files created

| File | Purpose |
|------|---------|
| `src/components/forms/ActionForms.tsx` | All 7 action modals |
| `src/components/wallet/WalletModals.tsx` | DepositModal + WithdrawModal |

## Files modified

`AuthScreen.tsx` · `AuthScreen.test.tsx` · `ProfileScreen.tsx` · `AgentReportScreen.tsx` · `market/AgentMarket.tsx` · `bourse/MerchantBourse.tsx` (+ all role screens from SP-01)
