# SM-02-02 — Investor fund action from FarmerDetailScreen

**Sprint:** SM-02 · Financing  
**Branch:** `feature/sm-02-financing`

## Context
`FarmerDetailScreen` shows farmer details and a funding progress bar. The "Financer" CTA exists but may not be wired to the real CF for creating a financing investment. This story ensures the full fund flow works: amount → confirm → success → balance update.

## Acceptance criteria
- [ ] "Financer" CTA on `FarmerDetailScreen` opens `PaymentModal` with `type="support"`
- [ ] On success, calls `httpsCallable(functions, "fundFarmer")` with `{ farmerId, amountUsd }`
- [ ] On success, invalidates `useInvestments` and `useFarmerDetail` queries
- [ ] Minimum funding amount: $10 (shown in PaymentModal subtitle)
- [ ] If insufficient wallet balance, shows error "Solde insuffisant" before payment screen
- [ ] Investor `InvestorFinancementScreen` shows updated funding progress after fund
- [ ] In devMode, `fundFarmer` CF call is skipped; success state shown immediately

## Implementation notes
- The web app uses `httpsCallable(functions, "fundFarmer")` — same CF for mobile
- Balance check: compare `amountUsd` against `userProfile.walletUsd` from AuthContext
