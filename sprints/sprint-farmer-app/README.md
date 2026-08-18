# Mombongo Farmer App — Sprint Plan

Dedicated React Native + Expo app for farmers (`mombongo-mobile` with EAS build profile `farmer`).
Locked to farmer role: no role selector, no investor/agent/merchant screens.
Firebase Auth + Cloud Functions backend — **no direct Firestore from the app**.

## Architecture decision

One Expo codebase; role-locked via `EXPO_PUBLIC_APP_ROLE=farmer` set in the EAS build profile.
The app reads this env var at startup; onboarding skips role selection and the tab bar shows only farmer screens.
Separate bundle ID (`com.mombongo.farmer`), app name (`Mombongo Agriculteur`), and icon.

## Sprint index

| Sprint | Folder | Focus | Status |
|--------|--------|-------|--------|
| SFA-00 | SFA-00-foundation | EAS profiles, role-lock auth, goal-first onboarding, FCM, profile | — |
| SFA-01 | SFA-01-core-screens | Dynamic home, crop calendar, market prices, exploitation, transactions, filter | — |
| SFA-02 | SFA-02-marketplace | Bourse publish + stepper, Pour vous, AgroExchange, videos, price chart | — |
| SFA-03 | SFA-03-financing | Financing apply + status, KYC upload, payments | — |
| SFA-04 | SFA-04-academia | Module player + XP, streaks + Mombongo Score | — |
| SFA-05 | SFA-05-retention | Status push, score unlock gates, offline cache, image caching + errors | — |
| SFA-06 | SFA-06-polish | Sentry + Analytics, iOS EAS build, CI/CD | — |
| SFA-07 | SFA-07-subscription | Subscription plans, premium content gating | — |

## Web → Mobile parity map

| Web sprint | Feature | Mobile sprint |
|------------|---------|---------------|
| SU-01-01 | Morning price push (CF already live) | SFA-00-04 (expo-notifications subscription) |
| SU-01-02 | Dynamic dashboard CTA | SFA-01-01 |
| SU-01-03 | Goal-first onboarding | SFA-00-03 |
| SU-01-04 | Mombongo Score widget | SFA-04-02 |
| SU-01-05 | Status push notifications | SFA-05-01 |
| SU-02-03 | Crop calendar widget | SFA-01-02 |
| SU-02-04 | Bourse progress stepper | SFA-02-01 |
| SU-02-05 | Academia streaks | SFA-04-02 |
| SU-03-01 | Revenue counter | SFA-01-01 |
| SU-03-02 | Bourse — Pour vous | SFA-02-02 |
| SU-03-04 | Score unlock thresholds | SFA-05-02 |
| SG-12 | Academia XP + progress | SFA-04-01 |
| SF-01–07 | Farm analytics, harvest, inputs, co-op | SFA-01-04 (exploitation base) |
| SM-04 | KYC document capture | SFA-03-02 |
| SM-06 | AgroExchange screen | SFA-02-03 |
| SM-08 | Stripe card + PawaPay polish | SFA-03-03 |
| SM-09 | Offline / React Query persistence | SFA-05-03 |
| SM-10 | Sentry + Analytics | SFA-06-01 |
| SM-11 | iOS EAS profiles | SFA-06-02 |

## Branch convention
- Feature: `feature/sfa-NN-slug` (e.g., `feature/sfa-00-foundation`)
- Patch: `feature/sfap-NN-slug`
- PRs merge into `dev`; `dev → main` triggers EAS preview build for `farmer` profile

## DONE convention
Append `-DONE` before `.md` when a story is fully verified.
`SFA-00-01-eas-build-profiles.md` → `SFA-00-01-eas-build-profiles-DONE.md`

## Success metrics (Farmer App)
- D1 retention ≥ 40% (vs ~15% baseline)
- D7 retention ≥ 25%
- Daily active farmers: 15–25 min/day across 3 touchpoints (morning price check, exploitation update, notification tap)
- Onboarding completion rate ≥ 80% (4-step flow)
- Push notification opt-in ≥ 70% (shown after onboarding completion)
