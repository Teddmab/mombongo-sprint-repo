# SU-01-03 — Goal-first onboarding flow

**Sprint:** SU-01 · Daily engagement  
**Branch:** `feature/su-01-daily-engagement`  
**Effort:** ~4 days

## Context
New farmers land on a cold dashboard after signup. This story adds a 3-step onboarding flow that runs once (first login only), captures their primary goal, does a minimal 3-field exploitation setup, then immediately delivers their first value: live prices for their crop and province.

## Flow

```
Signup complete
    ↓
OnboardingScreen (shown if user.onboardingComplete !== true)
    ↓
Step 1: Choose your goal (30 sec)
    • "Je veux vendre ma récolte"
    • "Je cherche un crédit agricole"  
    • "Je veux suivre et améliorer ma ferme"
    ↓
Step 2: Quick setup — 3 fields only (90 sec)
    • Nom de l'exploitation
    • Culture principale (dropdown: Maïs, Manioc, Arachide, Soja, Tomate, Autre)
    • Province (dropdown)
    → Calls httpsCallable 'createExploitation' with minimal payload
    ↓
Step 3: First value — price reveal (0 sec wait)
    • "Maïs dans votre province : 420–480 FC/kg aujourd'hui"
    • "3 acheteurs actifs en ce moment"
    • Reads from province_prices already on screen (pre-fetched before step 3 renders)
    ↓
Step 4: Score bar + next steps
    • "Votre profil Mombongo : 25/100"
    • 3 next actions shown as chips: "Ajouter la surface (+10 pts)", "KYC (+20 pts)", "Publier une annonce (+10 pts)"
    ↓
Dashboard (with onboardingComplete = true saved via CF)
```

## Implementation

### New screen: `src/pages/OnboardingScreen.tsx`
- Shown when `user.onboardingComplete !== true` (read from `useApp()` / AuthContext)
- Full-screen, no bottom nav, no top header
- Progress indicator: dots (4 steps)
- Back button on steps 2–4
- "Passer" (skip) link in corner — skips to dashboard, marks `onboardingComplete = true`

### Route
- Add `/onboarding` route in `src/App.tsx`
- Redirect from `/` to `/onboarding` when `!user.onboardingComplete && role === 'farmer'`

### Cloud Functions
- `completeOnboarding(payload: { goal: string, exploitationName: string, primaryCrop: string, province: string })` → creates exploitation doc + sets `users/{uid}.onboardingComplete = true` + `users/{uid}.primaryGoal = goal`
- Reuse existing `createExploitation` CF if it already accepts minimal fields

### Data model additions
```
users/{uid}/
  onboardingComplete: boolean   ← false by default
  primaryGoal: 'sell' | 'credit' | 'monitor'
```

## Acceptance criteria
- [ ] New farmer (first login) is redirected to `/onboarding` automatically
- [ ] Returning farmer (onboardingComplete = true) never sees onboarding again
- [ ] Goal selection persists to `users/{uid}.primaryGoal`
- [ ] Minimal exploitation is created after Step 2 (verify in Firestore)
- [ ] Step 3 price card shows real data from `province_prices` (or mock in devMode)
- [ ] "Passer" skip link works and marks onboarding complete
- [ ] Agent/investor/merchant roles do NOT trigger this flow
- [ ] `onboardingComplete = true` survives page refresh (not just in-memory state)

## Smoke test steps
1. Create a fresh farmer account → verify redirect to `/onboarding`
2. Complete all 4 steps → verify dashboard loads (no redirect loop)
3. Refresh page after completing onboarding → verify dashboard loads, not onboarding
4. Complete Step 2 → open Firebase Console → verify exploitation document created
5. Log in as existing farmer (who was created before this sprint) → verify NO onboarding redirect
6. Click "Passer" on step 1 → verify dashboard loads and onboardingComplete = true in Firestore
7. Test on mobile viewport — step cards must not overflow horizontally
