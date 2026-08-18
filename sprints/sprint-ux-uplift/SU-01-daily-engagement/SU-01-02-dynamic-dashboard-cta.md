# SU-01-02 — Dynamic priority CTA on farmer dashboard

**Sprint:** SU-01 · Daily engagement  
**Branch:** `feature/su-01-daily-engagement`  
**Effort:** ~2 days

## Context
The farmer dashboard currently shows 4 static quick-action buttons (Rapport, Exploitation, Prix marché, Formation) regardless of the farmer's actual state. This story replaces the hero section with a single, personalized "Aujourd'hui, faites ceci" card that changes based on what step the farmer is actually on.

## Priority logic (evaluated top-to-bottom, first match wins)

| Condition | CTA shown | Destination |
|-----------|-----------|-------------|
| No exploitation created | "Créez votre première exploitation" | `/exploitation` |
| KYC not submitted | "Complétez votre identité (KYC)" | `/profile/kyc` |
| KYC submitted, score < 40 | "Continuez votre formation pour débloquer la Bourse" | `/academia` |
| No active Bourse listing | "Publiez votre première annonce de vente" | `/bourse/new` |
| Active listing with 0 views | "Optimisez votre annonce pour attirer des acheteurs" | `/bourse` |
| Pending financing application | "Votre dossier de crédit est en cours — vérifiez le statut" | `/financement` |
| Score ≥ 60, no financing ever | "Vous êtes éligible à un crédit — postulez maintenant" | `/financement` |
| All above complete | "Bon travail ! Vérifiez les prix du marché ce matin" | `/market` |

## Implementation

### New hook: `useDashboardCTA` (`src/hooks/useDashboardCTA.ts`)
```typescript
// Returns { title, subtitle, icon, href, color }
// Reads from: useExploitations(), useKycStatus(), useMomBongoScore(), 
//             useFarmerListings(), useFinancingStatus()
// isDevMode() → returns mock CTA for current dev scenario
```

### UI changes in `AgricultorHome.tsx`
- Replace the green hero card (exploitation count) with `<DashboardCTACard />` component
- Keep the 4 quick-action buttons below (they become secondary, not primary)
- Mobile + Desktop variants both updated

### `DashboardCTACard` component (`src/components/dashboard/DashboardCTACard.tsx`)
- Large accent card with icon, title, subtitle, arrow button
- Color matches urgency: red (missing KYC), amber (incomplete profile), green (opportunity)
- Animated pulse on the arrow button (subtle, respects `html[data-slow]`)

## Acceptance criteria
- [ ] Farmer with no exploitation sees "Créez votre première exploitation" CTA
- [ ] Farmer with exploitation but no KYC sees KYC CTA
- [ ] Farmer with KYC and score ≥ 40 but no listing sees Bourse CTA
- [ ] CTA navigates correctly when tapped
- [ ] `isDevMode()` returns a deterministic mock CTA (no random)
- [ ] Desktop and mobile layouts both updated
- [ ] `html[data-slow]` disables card pulse animation

## Smoke test steps
1. Log in as a fresh farmer (no exploitation) → verify "Créez votre première exploitation" CTA
2. Create exploitation → verify CTA updates to KYC or Bourse prompt
3. Tap CTA → verify navigation lands on correct screen
4. Test on mobile viewport (375px) — card should fill width cleanly
5. Enable `VITE_DEV_MODE=true` — verify mock CTA renders without errors
