# SAG-01-03 — Agent Daily Visit Plan (Agent App)

## Context
Port of web sprint SU-02-02 (agent visit plan). The agent needs a daily list of which farmers to visit today, prioritized by:
1. Farmers with financing "awaiting_agent_action"
2. Farmers not visited in > 30 days
3. Farmers with upcoming harvest dates

The `getAgentVisitPlan` CF (planned in SU-02-02) returns a sorted list of farmers with a suggested visit reason and estimated time.

## Scope
- Create `src/hooks/useAgentVisitPlan.ts` — calls `getAgentVisitPlan` CF
- Add `VisitPlanCard` to `AgentHomeScreen.tsx` at the top (most important CTA)
- Each visit entry: farmer name, reason, suggested time, "En route" button
- "En route" button: opens Google Maps / Waze to farm GPS coords if available, otherwise copies address

## Cloud Function required
`getAgentVisitPlan` — planned in SU-02-02. Returns:
```typescript
{
  visits: Array<{
    farmerId: string
    farmerName: string
    reason: 'awaiting_action' | 'overdue_visit' | 'harvest_soon'
    reasonLabel: string
    suggestedTime: string // e.g. "09:00 – 10:30"
    gpsCoords?: { lat: number; lng: number }
    province: string
  }>
  totalEstimatedHours: number
}
```

## Files to create
- `src/hooks/useAgentVisitPlan.ts`
- `src/components/VisitPlanSection.tsx`

## Implementation

### `src/hooks/useAgentVisitPlan.ts`
```typescript
const MOCK_VISIT_PLAN = {
  visits: [
    { farmerId: 'f1', farmerName: 'Jean Kasongo', reason: 'awaiting_action', reasonLabel: 'Dossier en attente', suggestedTime: '09:00 – 10:30', province: 'Kinshasa' },
    { farmerId: 'f2', farmerName: 'Marie Lwamba', reason: 'overdue_visit', reasonLabel: 'Pas visité depuis 35 jours', suggestedTime: '11:00 – 12:00', province: 'Kinshasa' },
  ],
  totalEstimatedHours: 3.5,
}

export function useAgentVisitPlan() {
  return useQuery({
    queryKey: ['agentVisitPlan'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_VISIT_PLAN
      const res = await httpsCallable<void, typeof MOCK_VISIT_PLAN>(
        functions, 'getAgentVisitPlan'
      )()
      return res.data
    },
    staleTime: 60 * 60_000, // refresh once per hour
  })
}
```

### VisitPlanSection.tsx
```typescript
// Header: "Votre plan du jour · X.X heures de visites"
// List of visit cards, each with:
//   Farmer name + reason badge (color-coded by reason type)
//   Suggested time slot
//   "En route" button → opens maps app
//   "Terminer" button → marks visit done + prompts report submission
const openMaps = (coords?: { lat: number; lng: number }, name?: string) => {
  const url = coords
    ? `https://maps.google.com/?q=${coords.lat},${coords.lng}`
    : `https://maps.google.com/?q=${encodeURIComponent(name ?? '')}`
  Linking.openURL(url)
}
```

### AgentHomeScreen.tsx additions
```typescript
// Add VisitPlanSection above the quick action grid
const { data: visitPlan } = useAgentVisitPlan()
<VisitPlanSection plan={visitPlan} />
```

## Acceptance criteria
- [ ] Visit plan section appears at top of AgentHomeScreen
- [ ] Farmers sorted by urgency (awaiting_action first)
- [ ] "En route" opens Google Maps to farm coordinates or province name
- [ ] "Terminer" marks visit done and prompts report form
- [ ] Total estimated hours shown in section header
- [ ] Empty state: "Aucune visite planifiée pour aujourd'hui" when no farmers

## Smoke test
1. Open Agent home — confirm visit plan appears at top with mock visits
2. Tap "En route" → confirm Google Maps opens
3. Tap "Terminer" → confirm report form opens pre-filled with farmer
4. In live mode: confirm real visit plan loads from CF
