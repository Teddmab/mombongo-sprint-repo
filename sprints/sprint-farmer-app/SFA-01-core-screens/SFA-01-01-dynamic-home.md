# SFA-01-01 — Dynamic Farmer Home (DashboardCTA + Revenue Pill)

## Context
Port of web sprints SU-01-02 (DynamicDashboardCTA) and SU-03-01 (RevenueCounter) to mobile.

The current `FarmerHomeScreen.tsx` has:
- Hardcoded hero section with static "disbursed" / "target" values
- 4 static quick action buttons (always the same)
- `useCropTasks` and `useFarmerAlerts` from local mock only

This sprint replaces the static sections with real data from Cloud Functions.

## Scope
- Create `src/hooks/useDashboardCTA.ts` — calls `getDashboardCTA` CF
- Create `src/hooks/useFarmerRevenue.ts` — calls `getFarmerRevenue` CF
- Replace static hero card with `RevenuePill` showing real CDF totals
- Replace static quick actions with dynamic CTA card (same logic as web SU-01-02)
- Wire existing crop tasks / alerts sections to real `getCropTasks` CF

## Cloud Functions required
- `getDashboardCTA` — returns `{ ctaType, label, route, subtext }` (planned in SU-01-02; implement alongside)
- `getFarmerRevenue` — returns `{ totalRevenueCdf, periodLabel }` (planned in SU-03-01; implement alongside)
- `getCropTasks` — returns `{ tasks: CropTask[] }` (schedule reminders from exploitation)

## Files to modify
- `src/screens/home/FarmerHomeScreen.tsx` — replace hardcoded sections
- `src/hooks/useDashboardCTA.ts` — new
- `src/hooks/useFarmerRevenue.ts` — new
- `src/hooks/useCropTasks.ts` — convert from local mock to real CF call

## Implementation

### `src/hooks/useDashboardCTA.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

export type DashboardCTA = {
  ctaType: 'sell' | 'apply_financing' | 'track' | 'learn' | 'score'
  label: string
  route: string
  subtext?: string
}

const MOCK_CTA: DashboardCTA = {
  ctaType: 'sell',
  label: 'Publiez votre maïs sur la Bourse',
  route: 'Bourse',
  subtext: 'Prix actuel: 450 FC/kg',
}

export function useDashboardCTA() {
  return useQuery({
    queryKey: ['dashboardCTA'],
    queryFn: async (): Promise<DashboardCTA> => {
      if (isDevMode()) return MOCK_CTA
      const res = await httpsCallable<void, DashboardCTA>(functions, 'getDashboardCTA')()
      return res.data
    },
    staleTime: 10 * 60_000,
  })
}
```

### `src/hooks/useFarmerRevenue.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

type FarmerRevenue = { totalRevenueCdf: number; periodLabel: string }

export function useFarmerRevenue() {
  return useQuery({
    queryKey: ['farmerRevenue'],
    queryFn: async (): Promise<FarmerRevenue> => {
      if (isDevMode()) return { totalRevenueCdf: 850_000, periodLabel: 'ce mois' }
      const res = await httpsCallable<void, FarmerRevenue>(functions, 'getFarmerRevenue')()
      return res.data
    },
    staleTime: 30 * 60_000,
  })
}
```

### FarmerHomeScreen.tsx — key changes
```typescript
const { data: cta } = useDashboardCTA()
const { data: revenue } = useFarmerRevenue()

// Replace static hero:
<RevenuePill
  amount={revenue?.totalRevenueCdf ?? 0}
  label={revenue?.periodLabel ?? ''}
/>

// Replace static quick actions:
{cta && (
  <DashboardCTACard
    label={cta.label}
    subtext={cta.subtext}
    onPress={() => navigation.navigate(cta.route as never)}
  />
)}
```

## Acceptance criteria
- [ ] Home screen shows revenue total from `getFarmerRevenue` CF (not hardcoded)
- [ ] CTA card content changes based on farmer's state (sells when harvest ready, applies when eligible, etc.)
- [ ] Both values update after pull-to-refresh
- [ ] Dev mode shows mock values (no CF call)
- [ ] Loading state shows skeleton (not empty section)

## Smoke test
1. Sign in as farmer in dev mode — confirm mock revenue and mock CTA appear
2. Sign in as farmer in live mode (real Firebase) — confirm real CF data loads
3. Pull to refresh — data refetches without error
4. Tap CTA card — confirm navigation to correct screen
