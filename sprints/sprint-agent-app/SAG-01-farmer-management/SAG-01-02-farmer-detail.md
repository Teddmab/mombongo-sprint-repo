# SAG-01-02 — Agent Farmer Detail Screen (Real Data)

## Context
The mobile `FarmerDetailScreen.tsx` (in financing section) exists but shows financing-specific data. The agent needs a comprehensive farmer view: profile, exploitation summary, financing status, and quick actions (submit report, call, navigate to farm).

The `getAgentFarmerDetail` CF was deployed as part of SG-10 and returns:
- Farmer profile (name, province, cropType, phone, kycStatus)
- Exploitations summary (count, total area)
- Financing applications (current status)
- Last report summary (agent's last field report for this farmer)

## Scope
- Create `src/screens/farmers/AgentFarmerDetailScreen.tsx`
- Create `src/hooks/useAgentFarmerDetail.ts` — calls `getAgentFarmerDetail` CF
- Quick action buttons: Submit Report, Call Farmer, View Exploitation
- Show financing status timeline
- Show last report summary

## Cloud Functions required
- `getAgentFarmerDetail` — detailed farmer data for an agent (deployed in SG-10)

## Files to create
- `src/hooks/useAgentFarmerDetail.ts`
- `src/screens/farmers/AgentFarmerDetailScreen.tsx`

## Implementation

### `src/hooks/useAgentFarmerDetail.ts`
```typescript
export function useAgentFarmerDetail(farmerId: string) {
  return useQuery({
    queryKey: ['agentFarmerDetail', farmerId],
    queryFn: async () => {
      if (isDevMode()) return MOCK_FARMER_DETAIL
      const res = await httpsCallable<{ farmerId: string }, FarmerDetail>(
        functions, 'getAgentFarmerDetail'
      )({ farmerId })
      return res.data
    },
    enabled: !!farmerId,
  })
}
```

### AgentFarmerDetailScreen.tsx layout
```typescript
// Header: farmer name + avatar + province + crop chip + KYC status badge
// Quick actions row:
//   [📋 Rapport] [📞 Appeler] [🗺️ Itinéraire]
//   Report → ReportFormSheet with farmerId pre-filled
//   Call → Linking.openURL(`tel:${farmer.phone}`)
//   Route → Linking.openURL(maps URL to farm GPS coords if available)
// Sections:
//   Exploitation summary: count, area, crops
//   Financing timeline: same FinancingStatusCard as SFA-03-01
//   Last report: date, score, summary text
//   "Voir tous les rapports" link
```

## Acceptance criteria
- [ ] Farmer detail loads from CF with all sections populated
- [ ] "Appeler" button opens phone dialer with farmer's number
- [ ] "Rapport" button opens report form pre-filled with this farmer's ID
- [ ] Financing timeline shows correct current step
- [ ] Last report date and score shown
- [ ] Dev mode shows mock farmer detail

## Smoke test
1. Tap on a farmer in the farmer list → detail screen loads
2. Tap "Appeler" → phone dialer opens with correct number
3. Tap "Rapport" → report form opens with farmer pre-filled
4. Verify financing timeline step is correct
5. Tap "Voir tous les rapports" → report history filtered to this farmer
