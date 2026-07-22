# SM-07-00 — Subscription plans: real data + SubscriptionModal wiring

**Sprint:** SM-07 · Subscription  
**Branch:** `feature/sm-07-subscription`

## Context
`SubscriptionModal` component exists with hardcoded `SubscriptionPlan[]` data. The modal opens via the `PaymentModal` subscribe flow, but plans are not fetched from the backend and subscription status is not tracked.

## Acceptance criteria
- [ ] `useSubscriptionPlans()` hook: `httpsCallable(functions, "getSubscriptionPlans")` → `{ plans: SubscriptionPlan[] }`
- [ ] `SubscriptionModal` uses `useSubscriptionPlans()` instead of hardcoded plan array
- [ ] `useSubscriptionStatus()` hook: `httpsCallable(functions, "getSubscriptionStatus")` → `{ isActive, plan, expiresAt }`
- [ ] `ProfileScreen` shows active subscription card if subscribed (plan name, expiry date, "Gérer")
- [ ] On plan selection + payment success: `httpsCallable(functions, "activateSubscription")({ planId, transactionId })`
- [ ] `PaymentModal` `type="subscribe"` uses the planId from the selected plan
- [ ] After successful subscribe, `useSubscriptionStatus` query is invalidated
- [ ] In devMode, `isActive: false` returned (to always test the free-tier flow)

## Data shape
```ts
interface SubscriptionPlan {
  id: string;
  name: string;
  priceUsd: number;
  period: "monthly" | "annual";
  features: string[];
  isPremium: boolean;
  free?: boolean;
}
```

## Implementation notes
- Plans come from `platform_settings/subscription_plans` collection in Firestore (managed in admin)
- Subscription status stored in `users/{uid}.subscriptionPlan`, `.subscriptionExpiresAt`
