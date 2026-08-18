# Mombongo Agent App — Sprint Plan

Dedicated React Native + Expo app for field agents (`mombongo-mobile` with EAS build profile `agent`).
Locked to agent terrain role: no investor/farmer/merchant screens.
Firebase Auth + Cloud Functions backend — **no direct Firestore from the app**.

## Architecture decision

Same Expo codebase as Farmer App; role-locked via `EXPO_PUBLIC_APP_ROLE=agent` in the EAS build profile.
Separate bundle ID (`com.mombongo.agent`), app name (`Mombongo Agent`), icon.
The app skips role selection on auth and navigates only to agent screens.

## Sprint index

| Sprint | Folder | Focus | Status |
|--------|--------|-------|--------|
| SAG-00 | SAG-00-foundation | EAS profiles, role-lock auth, FCM setup | — |
| SAG-01 | SAG-01-farmer-management | Farmer list, farmer detail, visit plan, profile | — |
| SAG-02 | SAG-02-field-reports | Report form with photo upload, report history | — |
| SAG-03 | SAG-03-pipeline | Financing pipeline kanban, financing actions | — |
| SAG-04 | SAG-04-offline-monitoring | Offline cache, draft queue, Sentry, Analytics, iOS, image caching | — |
| SAG-05 | SAG-05-academia | Agent-specific courses, compliance gating | — |

## Web → Mobile parity map

| Web sprint | Feature | Mobile sprint |
|------------|---------|---------------|
| SU-02-02 | Agent daily visit plan | SAG-01-03 |
| SU-03-03 | Agent pipeline kanban | SAG-03-01 |
| SG-05 | Agent report wiring + history | SAG-02-01 |
| SG-10 | Agent screens real data | SAG-01-01, SAG-01-02 |
| SM-05 | Push notifications | SAG-00-03 |
| SM-09 | Offline / React Query persistence | SAG-04-01 |
| SM-10 | Sentry + Analytics | SAG-04-02 |
| SM-11 | iOS EAS profiles | SAG-04-02 |

## Branch convention
- Feature: `feature/sag-NN-slug` (e.g., `feature/sag-00-foundation`)
- Patch: `feature/sagp-NN-slug`
- PRs merge into `dev`; `dev → main` triggers EAS preview build for `agent` profile

## DONE convention
Append `-DONE` before `.md` when a story is fully verified.
`SAG-00-01-eas-build-profiles.md` → `SAG-00-01-eas-build-profiles-DONE.md`

## Success metrics (Agent App)
- Agent opens app on ≥ 80% of visit days
- Visit plan tapped within first 10 min of workday: ≥ 60%
- Field report submitted same day as visit: ≥ 75%
- Financing status updated within 24h of eligibility change: ≥ 90%
- Push notification opt-in: 100% (agents are staff — hard-require during onboarding)
