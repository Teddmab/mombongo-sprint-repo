# SM-02-01 — Agent terrain: farmer assignment & field visit scheduling

**Sprint:** SM-02 · Financing  
**Branch:** `feature/sm-02-financing`

## Context
Agent terrain users see a list of farmers assigned to them. The current `AgentFinancementScreen` shows farmers but doesn't support scheduling field visits or tracking completion status. The `AgentReportScreen` allows submitting a report but has no pre-filled context from a scheduled visit.

## Acceptance criteria
- [ ] `AgentFinancementScreen` has a "Visites planifiées" section showing upcoming visits
- [ ] Each visit shows: farmer name, date, status (planifiée / complétée / annulée)
- [ ] "Démarrer la visite" button on a planned visit navigates to `AgentReportScreen` with farmer pre-filled
- [ ] "Planifier une visite" button opens a bottom sheet to pick farmer + date
- [ ] `scheduleVisit({ farmerId, date })` calls `httpsCallable(functions, "scheduleAgentVisit")`
- [ ] Visit list comes from `httpsCallable(functions, "getAgentVisits")`
- [ ] `useAgentVisits()` hook in `hooks/useFinancing.ts`
- [ ] In devMode, returns mock visits from `data/mock.ts`

## Data shape
```ts
interface AgentVisit {
  id: string;
  farmerId: string;
  farmerName: string;
  scheduledDate: { seconds: number };
  status: "scheduled" | "completed" | "cancelled";
  reportId?: string;
}
```

## Implementation notes
- Date picker: use `expo-date-picker` or a simple text input (YYYY-MM-DD) for MVP
- Completed visits link to the submitted report if `reportId` is set
