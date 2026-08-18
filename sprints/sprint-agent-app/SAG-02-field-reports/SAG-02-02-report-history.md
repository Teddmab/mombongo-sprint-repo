# SAG-02-02 — Report History Screen (Agent App)

## Context
Agents need to review past field reports (their own submissions) to prepare for follow-up visits. The Rapports tab should show a filterable list of all submitted reports, with the ability to view detail and attached photos.

## Scope
- Create `src/hooks/useAgentReports.ts` — calls `getAgentReports` CF (replacing mock version)
- Create `src/screens/reports/AgentReportHistoryScreen.tsx`
- Create `src/screens/reports/AgentReportDetailScreen.tsx`
- Filter by farmer name or date range
- Show photo thumbnails in detail view

## Cloud Function required
`getAgentReports` — input: `{ farmerId?: string; limit?: number }` → output: `{ reports: AgentReport[] }`

The CF already exists from SG-05 web implementation.

## Files to create
- `src/hooks/useAgentReports.ts`
- `src/screens/reports/AgentReportHistoryScreen.tsx`
- `src/screens/reports/AgentReportDetailScreen.tsx`

## Implementation

### `src/hooks/useAgentReports.ts`
```typescript
export function useAgentReports(farmerId?: string) {
  return useQuery({
    queryKey: ['agentReports', farmerId],
    queryFn: async () => {
      if (isDevMode()) return MOCK_AGENT_REPORTS
      const res = await httpsCallable<{ farmerId?: string; limit?: number }, { reports: AgentReport[] }>(
        functions, 'getAgentReports'
      )({ farmerId, limit: 50 })
      return res.data.reports
    },
  })
}
```

### AgentReportHistoryScreen.tsx
```typescript
// Search bar: filter by farmer name
// Date filter chips: Aujourd'hui | Cette semaine | Ce mois
// Report list:
//   Each row: farmer name + visit date + crop health stars + photo count badge
//   Tap → AgentReportDetailScreen
// FAB: "Nouveau rapport" → AgentReportFormScreen
```

### AgentReportDetailScreen.tsx
```typescript
// Header: farmer name + visit date
// Crop health: star rating (1–5)
// Notes: full text
// Recommendations: full text
// Photos: horizontal scroll gallery of thumbnails
//   Tap thumbnail → full-screen photo viewer
// "Faire un suivi" button: open new report form pre-filled with this farmer
```

## Acceptance criteria
- [ ] Rapports tab shows list of reports from CF
- [ ] Filter by farmer name works
- [ ] Date filter chips filter correctly
- [ ] Report detail shows all fields including photos
- [ ] Photo thumbnails load (via `getDownloadUrl` signed URL from CF if needed)
- [ ] "Faire un suivi" opens new report form with farmer pre-filled

## Smoke test
1. Open Rapports tab — confirm reports list loads
2. Tap on a report — confirm detail screen shows all fields
3. Confirm photos visible (or placeholder if no photos)
4. Use search bar — confirm filtering works
5. Tap "Faire un suivi" — confirm new report form opens with correct farmer
