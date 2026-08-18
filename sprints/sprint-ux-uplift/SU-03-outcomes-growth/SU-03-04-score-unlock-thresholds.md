# SU-03-04 — Score-based unlock thresholds

**Sprint:** SU-03 · Outcomes & growth  
**Branch:** `feature/su-03-outcomes-growth`  
**Effort:** ~3 days

## Context
The Mombongo Score (SU-01-04) is visible on the dashboard, but it has no mechanical consequence yet. This story wires the score thresholds to actual feature access: Bourse requires score ≥ 40, Financement requires score ≥ 60. Below-threshold users see a clear, actionable lock screen explaining what they need to do.

## Threshold gates

| Feature | Required Score | Lock message |
|---------|---------------|-------------|
| Bourse (publish listing) | 40 | "Atteignez le score 40 pour publier sur la Bourse" |
| Financement (apply for credit) | 60 | "Atteignez le score 60 pour postuler à un crédit" |
| Taux préférentiel (shown in financement) | 80 | "Score 80 — Taux préférentiel appliqué ✓" |

Note: **viewing** prices and listings is always available. The gate only applies to **publishing/applying**.

## Implementation

### Gate component: `ScoreGate` (`src/components/ScoreGate.tsx`)
```tsx
<ScoreGate requiredScore={40} currentScore={score} featureName="la Bourse">
  {/* children render only when score >= requiredScore */}
</ScoreGate>
```

When score is below threshold, renders instead:
- Lock icon + "Score requis : 40 / 100"
- Progress bar showing current score toward threshold
- "Il vous manque X points" 
- 3 fastest ways to gain those points (from the breakdown — e.g., "Compléter KYC +20 pts")
- "Commencer →" CTA pointing to the fastest path

### Placement
- **Bourse**: wrap the "Nouvelle annonce" button and the "Pré-acheter" form in `<ScoreGate requiredScore={40}>`
- **Financement**: wrap the "Déposer une demande" form entry in `<ScoreGate requiredScore={60}>`

### Hook: extend `useMomBongoScore` (SU-01-04)
- Return `unlockedFeatures: Set<'bourse' | 'financement' | 'preferred_rate'>`
- Based on score thresholds above

### devMode behavior
- `isDevMode()` with score mock of 45 → Bourse unlocked, Financement locked
- A `?mockScore=25` URL param overrides the dev mock score for testing locked states

## Acceptance criteria
- [ ] Farmer with score < 40: "Nouvelle annonce" button replaced by ScoreGate lock state
- [ ] Farmer with score 40–59: Bourse publishing unlocked; Financement still locked
- [ ] Farmer with score ≥ 60: Financement access unlocked
- [ ] Lock screen shows current score, threshold, points missing, and 3 fastest actions
- [ ] "Commencer →" CTA navigates to the correct screen to gain points
- [ ] Preferred rate badge shown in Financement when score ≥ 80
- [ ] `?mockScore=25` URL param works in devMode to test locked states
- [ ] Score gate never blocks **reading** listings — only publishing/applying

## Smoke test steps
1. In devMode, open `?mockScore=25` → go to Bourse → verify "Nouvelle annonce" is replaced by lock state
2. Lock screen shows "Score requis : 40 / 100" and "Il vous manque 15 points"
3. Verify 3 actions listed with correct point values
4. Tap "Commencer →" on first action → verify navigation
5. Open `?mockScore=45` → Bourse unlocked; open Financement → still locked (score 60 required)
6. Open `?mockScore=80` → Financement unlocked; verify preferred rate badge visible
7. In devMode (score = 45): verify listing cards are still readable (browsing not gated)
