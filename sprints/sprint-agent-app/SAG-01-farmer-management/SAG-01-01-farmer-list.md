# SAG-01-01 — Agent Farmer List (Real Data)

## Context
Port of web sprint SG-10 (Agent screens real data). The current mobile `AgentHomeScreen.tsx` uses `useAgentFarmers` from `useLocalData` — a local mock. This sprint wires it to the `getAgentFarmers` CF and creates the dedicated `AgentFarmerListScreen.tsx` with urgency-sorted farmers.

Urgency sort: farmers with overdue financing actions come first (status = "awaiting_agent_action"), then farmers with pending reports, then others.

The `getAgentFarmers` CF already exists (deployed in SG-10).

## Scope
- Create `src/hooks/useAgentFarmers.ts` — real CF call (replacing local mock version)
- Create `src/screens/farmers/AgentFarmerListScreen.tsx`
- Add urgency badge on farmer cards (red dot for "awaiting_agent_action")
- Pull-to-refresh
- Search bar (filter by name or crop type)

## Cloud Functions required
- `getAgentFarmers` — returns list of farmers assigned to the agent, with their status and last visit date

## Files to create / modify
- `src/hooks/useAgentFarmers.ts` — real version
- `src/screens/farmers/AgentFarmerListScreen.tsx`

## Implementation

### `src/hooks/useAgentFarmers.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'
import { MOCK_AGENT_FARMERS } from '@/data/mock'

export type AgentFarmer = {
  uid: string
  name: string
  province: string
  cropType: string
  status: 'active' | 'awaiting_agent_action' | 'pending_review' | 'funded'
  lastVisitDate?: string
  financingApplicationId?: string
}

export function useAgentFarmers() {
  return useQuery({
    queryKey: ['agentFarmers'],
    queryFn: async (): Promise<AgentFarmer[]> => {
      if (isDevMode()) return MOCK_AGENT_FARMERS
      const res = await httpsCallable<void, { farmers: AgentFarmer[] }>(
        functions, 'getAgentFarmers'
      )()
      return res.data.farmers
    },
  })
}
```

### AgentFarmerListScreen.tsx
```typescript
// Search bar at top
// Sorted: awaiting_agent_action first (red badge), then pending_review (yellow), then active
// Each farmer card:
//   Name + province chip + crop chip
//   Status badge (color-coded)
//   Last visit: "Il y a 3 jours" or "Jamais visité"
//   Tap → AgentFarmerDetailScreen
// FAB: none (farmers are assigned by admin, not added by agent)
```

## Acceptance criteria
- [ ] Farmer list loads from `getAgentFarmers` CF
- [ ] Farmers sorted by urgency (awaiting_agent_action at top)
- [ ] Red badge on "awaiting_agent_action" farmers
- [ ] Search filters by name and crop type correctly
- [ ] Pull-to-refresh works
- [ ] Dev mode shows 5+ mock farmers with different statuses

## Smoke test
1. Open Farmers tab — confirm list loads
2. Verify at least one "awaiting_agent_action" farmer appears at top with red badge
3. Type in search bar → confirm list filters in real-time
4. Pull to refresh → no crash, list updates
