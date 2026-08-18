# SU-01-04 — Mombongo Score widget on farmer dashboard

**Sprint:** SU-01 · Daily engagement  
**Branch:** `feature/su-01-daily-engagement`  
**Effort:** ~5 days (2 CF + 2 frontend + 1 test)

## Context
A single number (0–100) visible permanently on the farmer dashboard. It progresses as the farmer completes meaningful actions and unlocks features at defined thresholds. It gives the farmer a progress goal and communicates what Mombongo expects from them.

## Score calculation (server-side)

| Action | Points | Source |
|--------|--------|--------|
| Profile complete (name + photo + phone) | +20 | `users/{uid}` |
| KYC approved | +20 | `users/{uid}.kycStatus === 'approved'` |
| Exploitation active (surface + culture) | +10 | `exploitations` collection |
| First Bourse listing published | +10 | `listings` collection |
| Each Academia course completed | +5 (max 15) | `courseProgress` collection |
| Price alert activated | +5 | `users/{uid}.priceAlerts` non-empty |
| First Bourse transaction completed | +15 | `transactions` collection |
| Active loan (financing approved) | +10 | `financingApplications` collection |

**Max total: 105 pts → capped at 100**

## Unlock thresholds

| Score | Unlocks |
|-------|---------|
| 0–39 | Prix marché + Academia |
| 40 | Accès Bourse (publish + buy) |
| 60 | Accès Financement (apply for credit) |
| 80 | Taux préférentiel flag (shown in financement) |
| 100 | "Agriculteur Certifié Mombongo" badge |

## Implementation

### Cloud Function: `calculateMomBongoScore(uid)`
- Triggered: on any write to `users/{uid}`, `exploitations`, `listings`, `transactions`, `financingApplications`, `courseProgress`
- Or: callable on demand `getMomBongoScore()` → returns `{ score, breakdown, unlockedAt }` 
- Stores result in `users/{uid}.momBongoScore` (number) + `users/{uid}.momBongoBreakdown` (map)

### New hook: `useMomBongoScore` (`src/hooks/useMomBongoScore.ts`)
```typescript
// Returns { score: number, breakdown: ScoreBreakdown, isLoading: boolean }
// isDevMode() → returns { score: 45, breakdown: {...}, isLoading: false }
// Real path: httpsCallable(functions, 'getMomBongoScore')()
```

### UI: `ScoreWidget` (`src/components/dashboard/ScoreWidget.tsx`)
- Circular progress ring (SVG, no external lib) showing `score / 100`
- Number in center: bold, large
- Below ring: "Niveau X / 4" where level = threshold tier
- "Prochain déblocage : score 60 → Accès crédit" — shows next milestone
- Tapping the widget opens a `ScoreBreakdownSheet` (bottom sheet on mobile, modal on desktop)
- Added to `AgricultorHome.tsx` below the DashboardCTACard (SU-01-02)

### `ScoreBreakdownSheet`
- Lists each score component with current state (earned / not earned)
- Tap on an unearned item → navigates to the relevant screen

## Acceptance criteria
- [ ] `getMomBongoScore` CF returns correct score for a farmer with known state
- [ ] Score updates within 60 seconds after a qualifying action (e.g., completing a course)
- [ ] `ScoreWidget` renders on both mobile and desktop farmer dashboard
- [ ] Circular progress ring animates from 0 to current score on first render (respects `data-slow`)
- [ ] Tapping the widget opens the breakdown sheet
- [ ] Each breakdown item links to the correct screen
- [ ] Score is cached by React Query (staleTime: 5 min) — no waterfall on every render
- [ ] `isDevMode()` returns score 45 deterministically

## Smoke test steps
1. Log in as farmer with only exploitation created → verify score shows ~10 pts
2. Complete a course → wait 60 sec → refresh → verify score increased by 5
3. Submit KYC → wait for approval → verify score jumps by 20
4. Tap the score widget → breakdown sheet opens, shows each component
5. Tap an unearned item (e.g., "Publier une annonce") → navigates to Bourse
6. On slow connection (DevTools → Network → Slow 3G) → verify ring animation is instant (data-slow)
