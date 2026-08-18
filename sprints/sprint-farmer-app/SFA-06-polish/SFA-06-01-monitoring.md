# SFA-06-01 — Sentry + Analytics (Farmer App)

## Context
Port of SM-10. The web has Sentry + Firebase Analytics wired. Mobile needs the same so crashes are visible and key business events are tracked.

## Scope
- Install and configure Sentry React Native (`@sentry/react-native`)
- Install Firebase Analytics (`@react-native-firebase/analytics` or via `expo-firebase-analytics`)
- Track key farmer events: `onboarding_complete`, `module_complete`, `harvest_recorded`, `listing_published`, `financing_applied`, `payment_initiated`
- Wrap root component in `Sentry.wrap()`

## Install commands
```bash
npx expo install @sentry/react-native
npx expo install expo-firebase-analytics
```

## Files to modify
- `App.tsx` — Sentry init + wrap
- `src/lib/analytics.ts` — typed event tracking helpers
- Call tracking in relevant screens/hooks

## Implementation

### `src/lib/analytics.ts`
```typescript
import * as Analytics from 'expo-firebase-analytics'

export const track = {
  onboardingComplete: (p: { cropType: string; province: string; goal: string }) =>
    Analytics.logEvent('onboarding_complete', p),
  moduleComplete: (p: { moduleId: string; courseId: string; xpEarned: number }) =>
    Analytics.logEvent('module_complete', p),
  harvestRecorded: (p: { exploitationId: string; quantityKg: number; pricePerKgCdf: number }) =>
    Analytics.logEvent('harvest_recorded', p),
  listingPublished: (p: { commodity: string; quantityKg: number }) =>
    Analytics.logEvent('listing_published', p),
  financingApplied: (p: { amountCdf: number }) =>
    Analytics.logEvent('financing_applied', p),
  paymentInitiated: (p: { method: 'pawapay' | 'card'; amountCdf: number }) =>
    Analytics.logEvent('payment_initiated', p),
}
```

## Acceptance criteria
- [ ] Sentry DSN configured; test crash appears in Sentry dashboard
- [ ] `onboarding_complete` event fires with correct props after step 4
- [ ] `module_complete` fires with xpEarned after ModulePlayer finishes
- [ ] `listing_published` fires after CreateListingSheet submit
- [ ] No PII in event properties (no names, phone numbers, emails)

## Smoke test
1. Trigger a test crash → verify in Sentry dashboard
2. Complete onboarding → open Firebase console → DebugView → confirm `onboarding_complete` event
3. Complete a module → confirm `module_complete` event with xpEarned > 0
