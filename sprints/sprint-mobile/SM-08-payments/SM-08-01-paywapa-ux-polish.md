# SM-08-01 — PawaPay / mobile money UX polish

**Sprint:** SM-08 · Payments  
**Branch:** `feature/sm-08-payments`

## Context
`WalletModals` deposit/withdraw flow calls real Cloud Functions but the UX needs polish: better status polling, clearer error states, operator logos, and a real timeout.

## Acceptance criteria
- [ ] Operator logos: show actual M-Pesa, Airtel Money, Orange Money brand colors and icons (already partially done — verify)
- [ ] Phone validation: Congolese format (`+243` prefix, 9 digits after) with real-time error
- [ ] Status polling: after initiating deposit/withdraw, poll `getDepositStatus`/`getWithdrawStatus` every 5s for up to 2 minutes
- [ ] If status = "pending" after 2 minutes: show "En attente de confirmation de l'opérateur" + link to check later
- [ ] If status = "failed": show failure reason from CF response
- [ ] "Annuler" available while polling (calls `cancelDeposit` CF if it exists, else just closes)
- [ ] After success: play haptic feedback (`expo-haptics` lightImpact) + show success animation
- [ ] Minimum deposit: $1 (shown below amount field)
- [ ] Maximum withdraw: wallet balance (validated before submit)
- [ ] Wallet balance refreshed after successful deposit/withdrawal via `queryClient.invalidateQueries`

## Implementation notes
- `expo-haptics` is likely in Expo SDK — verify and add to imports
- Polling: use `setInterval` stored in a `useRef`, cleared on modal close or on terminal status
- The `cancelDeposit` CF may not exist yet — if not, just allow close without cancel call
