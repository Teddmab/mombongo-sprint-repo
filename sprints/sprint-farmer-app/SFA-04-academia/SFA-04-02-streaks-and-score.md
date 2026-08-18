# SFA-04-02 — Academia Streaks + Mombongo Score (Farmer App)

## Context
Port of web sprints SU-02-05 (academia streaks) and SU-01-04 (Mombongo Score). These are retention mechanics: the streak encourages daily learning; the score summarizes the farmer's overall platform engagement.

CFs to implement (planned in web, same CFs work for mobile):
- `getAcademiaStreak` — returns `{ currentStreak, longestStreak, lastActivityDate, streakAtRisk }`
- `getMomBongoScore` — returns `{ score, breakdown, level }`

## Scope
- Create `src/hooks/useAcademiaStreak.ts`
- Create `src/hooks/useMomBongoScore.ts`
- Add `StreakWidget` to AcademiaScreen header
- Add `ScoreWidget` to FarmerHomeScreen (below revenue pill)
- Streak-at-risk push: the `sendStreakReminderPush` CF (SU-02-05) sends a push if no activity by 20:00 — mobile just needs to handle the tap (navigate to Academia)

## Cloud Functions required
- `getAcademiaStreak` — planned in SU-02-05
- `getMomBongoScore` — planned in SU-01-04

## Files to create
- `src/hooks/useAcademiaStreak.ts`
- `src/hooks/useMomBongoScore.ts`
- `src/components/StreakWidget.tsx`
- `src/components/ScoreWidget.tsx` (SVG ring, same visual as web but RN SVG)

## Implementation

### `src/hooks/useAcademiaStreak.ts`
```typescript
const MOCK_STREAK = { currentStreak: 5, longestStreak: 12, lastActivityDate: new Date().toISOString(), streakAtRisk: false }

export function useAcademiaStreak() {
  return useQuery({
    queryKey: ['academiaStreak'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_STREAK
      const res = await httpsCallable<void, typeof MOCK_STREAK>(functions, 'getAcademiaStreak')()
      return res.data
    },
    staleTime: 30 * 60_000,
  })
}
```

### `src/hooks/useMomBongoScore.ts`
```typescript
type ScoreData = {
  score: number // 0-100
  level: 'débutant' | 'intermédiaire' | 'avancé' | 'expert'
  breakdown: { label: string; points: number; max: number }[]
}

const MOCK_SCORE: ScoreData = {
  score: 62,
  level: 'intermédiaire',
  breakdown: [
    { label: 'Activité', points: 20, max: 25 },
    { label: 'Transactions', points: 15, max: 25 },
    { label: 'Formation', points: 18, max: 25 },
    { label: 'Profil complet', points: 9, max: 25 },
  ],
}

export function useMomBongoScore() {
  return useQuery({
    queryKey: ['momBongoScore'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_SCORE
      const res = await httpsCallable<void, ScoreData>(functions, 'getMomBongoScore')()
      return res.data
    },
    staleTime: 60 * 60_000,
  })
}
```

### StreakWidget.tsx (React Native)
```typescript
import Animated, { useSharedValue, withSpring } from 'react-native-reanimated'

// Flame icon + streak count
// "🔥 5 jours · Série en cours"
// Red "⚠️ À risque" badge if streakAtRisk === true
```

### ScoreWidget.tsx (RN SVG ring)
```typescript
import Svg, { Circle, Text as SvgText } from 'react-native-svg'

// Circular progress ring (same concept as web SVG)
// Center: score number + "/ 100"
// Below: level badge
// Tap → navigate to a ScoreDetailScreen (future) or show breakdown modal
```

## Install command
```bash
npx expo install react-native-svg
```

## Acceptance criteria
- [ ] StreakWidget in AcademiaScreen header shows current streak and at-risk state
- [ ] ScoreWidget in FarmerHome shows score (0-100) as circular ring
- [ ] Both widgets show dev mode mock values
- [ ] Completing a module invalidates streak query (streak increments)
- [ ] Streak-at-risk push tap navigates to Academia tab

## Smoke test
1. Open FarmerHome — confirm ScoreWidget ring shows 62 (mock) or real score
2. Open Academia — confirm StreakWidget shows "🔥 5 jours"
3. Complete a module — return to Academia — confirm streak count updated
4. From Firebase console send push with `{ "screen": "academia" }` → tap → confirm navigates to Academia tab
