# Mombongo Web — UX Uplift Sprint Plan

Transforms the app from an "annuaire" (form-filling directory) into a purpose-driven agricultural companion with daily usage pull.

Rooted in the analysis document at `mombongo-sprint-repo/ux-strategy.md` and the artifact published at https://claude.ai/code/artifact/98527ad3-17e0-4ae7-aaa5-a276ef4f4825.

## Sprint index

| Sprint | Folder | Theme | Estimated effort |
|--------|--------|-------|-----------------|
| SU-01 | SU-01-daily-engagement | Daily hooks — push, dynamic dashboard, onboarding, score | ~14 days |
| SU-02 | SU-02-guided-journey | Journey clarity — videos, agent plan, crop calendar, bourse stepper, streaks | ~15 days |
| SU-03 | SU-03-outcomes-growth | Outcomes — revenue counter, bourse matching, agent kanban, score gates | ~12 days |

## Branch convention
- Feature: `feature/su-NN-slug` (e.g., `feature/su-01-daily-engagement`)
- Patch: `feature/sup-NN-slug`
- PRs merge into `dev`; `dev → main` requires manual approval

## DONE convention
When a story is fully implemented and smoke-tested, rename the file by appending `-DONE` before `.md`.
`SU-01-01-morning-price-push.md` → `SU-01-01-morning-price-push-DONE.md`

## Architecture rules (unchanged)
- Frontend never calls Firestore directly — all data via `httpsCallable(functions, 'name')(payload)`
- New Cloud Functions go in `mombongo-functions/src/index.ts`
- Push notifications via FCM — token stored in `users/{uid}/fcmToken` document (via CF)
- `isDevMode()` guard on every new hook — mock path first, real CF path behind it

## Success metrics (to measure after SU-03)
- DAU/MAU ratio: target > 35% (currently ~5%)
- D7 retention: target > 40%
- Average session duration: target > 8 min
- Morning push open rate: target > 25%
- Score 40+ users (Bourse unlocked): target > 60% of active users
