# SFA-01-02 — Crop Calendar Widget (Farmer App)

## Context
Port of web sprint SU-02-03. The crop calendar shows the current phase of the farmer's primary crop (sowing, growing, harvesting, storage) as a horizontal timeline with today marked. The calendar is computed from `cropType` + `province` + planting date.

## Scope
- Create `src/hooks/useCropCalendar.ts` — calls `getCropCalendar` CF
- Create `src/components/CropCalendarWidget.tsx` — horizontal phase bar
- Add widget to `FarmerHomeScreen.tsx` below the revenue pill

## Cloud Function required
`getCropCalendar` — input: `{ cropType, province }` → output:
```typescript
{
  phases: Array<{ name: string; startDay: number; endDay: number; emoji: string }>,
  currentPhase: string,
  daysInPhase: number,
  nextPhase: string,
  daysToNextPhase: number,
}
```

## Files to create
- `src/hooks/useCropCalendar.ts`
- `src/components/CropCalendarWidget.tsx`

## Implementation

### `src/hooks/useCropCalendar.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

const MOCK_CALENDAR = {
  phases: [
    { name: 'Préparation', startDay: 0, endDay: 14, emoji: '🌱' },
    { name: 'Croissance', startDay: 14, endDay: 60, emoji: '🌿' },
    { name: 'Floraison', startDay: 60, endDay: 90, emoji: '🌸' },
    { name: 'Récolte', startDay: 90, endDay: 120, emoji: '🌾' },
  ],
  currentPhase: 'Croissance',
  daysInPhase: 12,
  nextPhase: 'Floraison',
  daysToNextPhase: 34,
}

export function useCropCalendar() {
  return useQuery({
    queryKey: ['cropCalendar'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_CALENDAR
      const res = await httpsCallable<void, typeof MOCK_CALENDAR>(
        functions, 'getCropCalendar'
      )()
      return res.data
    },
    staleTime: 60 * 60_000, // changes daily at most
  })
}
```

### `src/components/CropCalendarWidget.tsx`
```typescript
export function CropCalendarWidget({ calendar }: { calendar: ReturnType<typeof useCropCalendar>['data'] }) {
  if (!calendar) return <Skeleton height={80} />
  return (
    <View style={styles.container}>
      <Text style={styles.label}>Phase actuelle · {calendar.currentPhase}</Text>
      <Text style={styles.sub}>
        {calendar.daysToNextPhase}j jusqu'à {calendar.nextPhase}
      </Text>
      <View style={styles.phaseBar}>
        {calendar.phases.map(phase => (
          <PhaseSegment
            key={phase.name}
            phase={phase}
            isActive={phase.name === calendar.currentPhase}
          />
        ))}
      </View>
    </View>
  )
}
```

## Acceptance criteria
- [ ] Widget renders in FarmerHomeScreen with correct phase for the farmer's cropType
- [ ] Active phase is visually highlighted (different color or indicator)
- [ ] "X jours jusqu'à [next phase]" shown correctly
- [ ] Skeleton shown while loading
- [ ] Dev mode shows mock data

## Smoke test
1. Open FarmerHome — confirm crop calendar widget renders
2. Verify active phase label matches expected phase for the mock cropType + date
3. In live mode with a farmer who has `cropType: 'maize'` — verify real calendar data loads
