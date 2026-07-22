# SM-10-01 — Firebase Analytics: screen_view + business events

**Sprint:** SM-10 · Monitoring  
**Branch:** `feature/sm-10-monitoring`

## Context
The web app tracks Analytics events for key actions (investments, bourse activity, KYC completion). Mobile has Firebase configured but Analytics events are not being logged, meaning we have no visibility into user behavior.

## Acceptance criteria

### Screen tracking
- [ ] `analytics.logEvent("screen_view", { screen_name, screen_class })` called on every tab navigation
- [ ] Implemented via `useFocusEffect` in each tab screen + stack screen `useLayoutEffect`
- [ ] Helper: `useScreenTracking(screenName)` hook in `hooks/useAnalytics.ts`

### Business events to track

| Event | Trigger | Parameters |
|-------|---------|------------|
| `investment_initiated` | PaymentModal opens with `type="invest"` | `{ productId, amountUsd }` |
| `investment_completed` | PaymentModal success step | `{ productId, amountUsd, method }` |
| `deposit_initiated` | WalletModal deposit submit | `{ amountUsd, operator }` |
| `deposit_completed` | Deposit status = completed | `{ amountUsd, operator }` |
| `kyc_submitted` | KYC submitKyc CF called | `{ docType }` |
| `kyc_verified` | `useKycStatus` returns `verified` | — |
| `course_enrolled` | enrollCourse CF called | `{ courseId, courseName }` |
| `module_completed` | completeModule CF success | `{ courseId, moduleId, moduleType }` |
| `certificate_earned` | courseCompleted = true | `{ courseId }` |
| `subscription_activated` | activateSubscription CF success | `{ planId, priceUsd }` |

- [ ] All events above logged via `analytics.logEvent(eventName, params)`
- [ ] Analytics disabled in devMode (check `isDevMode()` before logging)
- [ ] `useAnalytics.ts` hook: `const { logEvent } = useAnalytics()` (wraps Firebase analytics)

## Implementation notes
- Firebase Analytics is already in the Firebase config — just import `getAnalytics(app)` and `logEvent`
- In Expo managed workflow, Analytics works without native modules for basic events
- Test via Firebase DebugView: `adb shell setprop debug.firebase.analytics.app com.mombongo.mobile`
