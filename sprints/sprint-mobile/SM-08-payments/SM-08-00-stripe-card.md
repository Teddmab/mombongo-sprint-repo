# SM-08-00 — Stripe card payment path in PaymentModal

**Sprint:** SM-08 · Payments  
**Branch:** `feature/sm-08-payments`

## Context
`PaymentModal` has a "method" step with two options: "mobile money" and "carte bancaire". The mobile money path works (via `wallet.service.ts`). The card path currently shows a stub (no Stripe integration). This story wires Stripe for card payments.

## Acceptance criteria
- [ ] "Carte bancaire" method opens a card input step with: card number, expiry, CVC fields
- [ ] On "Payer", calls `httpsCallable(functions, "createStripePaymentIntent")({ amountUsd, type, metadata })` → `{ clientSecret }`
- [ ] Uses `@stripe/stripe-react-native` SDK to confirm the payment with `confirmPayment(clientSecret, { paymentMethodType: "Card", paymentMethodData: { billingDetails } })`
- [ ] On success: show the success step (same as mobile money success)
- [ ] On failure: show error message + "Réessayer" button
- [ ] Stripe publishable key: loaded from `expo-constants` (`EXPO_PUBLIC_STRIPE_PUBLISHABLE_KEY` env var)
- [ ] `StripeProvider` wraps the app in `context/Providers.tsx` with the publishable key

## Implementation notes
- `@stripe/stripe-react-native` must be added to package.json and babel.config.js
- Managed workflow: Stripe RN requires EAS build (native module) — cannot use Expo Go
- Dev mode: skip Stripe call; show success immediately
- The Cloud Function `createStripePaymentIntent` must already exist in mombongo-functions (check before implementing)

## EAS build note
After adding `@stripe/stripe-react-native`, all dev/preview/production builds must go through EAS (not Expo Go). Update README to reflect this.
