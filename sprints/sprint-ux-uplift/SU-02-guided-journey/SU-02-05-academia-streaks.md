# SU-02-05 — Academia streaks + daily learning goal widget

**Sprint:** SU-02 · Guided journey  
**Branch:** `feature/su-02-guided-journey`  
**Effort:** ~3 days

## Context
Academia has courses but no reason to return daily. This story adds a streak counter (consecutive days with at least one course completed), a daily learning goal on the dashboard, and a push reminder when a streak of 2+ days is in danger of being broken.

## Streak model
- A "day" is counted when the user completes at least one course module during that calendar day (Africa/Kinshasa timezone)
- Streak resets if a day is skipped (no completion)
- Streak is stored in `users/{uid}.academiaStreak: { count: number, lastCompletedDate: string }`

## Implementation

### CF updates
- `onCourseCompleted` trigger (already fires when a module is marked done): 
  - Read `users/{uid}.academiaStreak`
  - If `lastCompletedDate === yesterday`: increment `count`, update `lastCompletedDate`
  - If `lastCompletedDate === today`: do nothing (already counted today)
  - If older: reset count to 1, update `lastCompletedDate`
  - Write back to `users/{uid}.academiaStreak`

- New scheduled CF `sendAcademiaStreakReminder` (runs daily at 18:00 WAT):
  - Finds users where `academiaStreak.count >= 2` AND `lastCompletedDate === yesterday`
  - Sends push: "Votre série de {count} jours est en danger ! Complétez un cours maintenant."
  - Tap: opens Academia screen

### UI changes

#### Academia screen header
- Add streak badge: "🔥 {count} jours" in top-right of screen header
- On first completion of the day: toast animation "+1 à votre série"
- Badge tooltip: "Série consécutive de cours complétés"

#### Dashboard: "Objectif du jour" widget (below ScoreWidget)
- "📚 Formation du jour : 10 min" with a progress ring (0/1 module completed today)
- After first module today: ring fills, label "Objectif du jour atteint ✓"
- Tapping the widget navigates to Academia

#### Academia course cards
- Each card shows "+5 pts Score" badge in corner (from SU-01-04 integration)
- "Nouveau !" badge on courses added in the last 7 days

### New hook: `useAcademiaStreak` (`src/hooks/useAcademiaStreak.ts`)
```typescript
// Returns { streakCount: number, completedToday: boolean, isLoading: boolean }
// isDevMode() → { streakCount: 4, completedToday: false }
```

## Acceptance criteria
- [ ] Streak count increments correctly after completing a module (verified next day or via CF test)
- [ ] Streak resets to 1 after a skipped day
- [ ] Streak badge visible on Academia screen header
- [ ] Dashboard "Objectif du jour" widget visible; fills when first module completed
- [ ] Push reminder fires at 18:00 WAT for users with streak ≥ 2 at risk
- [ ] "+5 pts Score" badge visible on each course card
- [ ] `isDevMode()` returns streak 4, completedToday false

## Smoke test steps
1. Complete a course module → verify toast "+1 à votre série" appears
2. Check Academia header → verify streak badge shows "🔥 1 jour"
3. Check dashboard → verify "Objectif du jour" ring is filled
4. Manually trigger `sendAcademiaStreakReminder` CF → verify push on test device
5. Verify "+5 pts Score" badge visible on all course cards
6. Complete second module same day → verify streak still shows 1 (not double-counted)
