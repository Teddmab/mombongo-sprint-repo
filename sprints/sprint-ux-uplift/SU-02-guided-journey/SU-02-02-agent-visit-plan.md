# SU-02-02 — Agent daily visit plan section

**Sprint:** SU-02 · Guided journey  
**Branch:** `feature/su-02-guided-journey`  
**Effort:** ~3 days

## Context
The agent currently sees a list of all farmers sorted by urgency, but has no structured "plan for today." This story adds a "Plan de visite du jour" section at the top of the agent dashboard that surfaces 3 prioritized farmers to visit, with the reason for each visit.

## Priority algorithm (CF-computed)

A farmer appears in today's plan if ANY of the following is true, in this priority order:
1. **Urgence crédit** — a financing application in `under_review` status for > 5 days with no agent action recorded
2. **Visite programmée** — `nextVisitDate` on the farmer's profile falls within today ± 1 day
3. **Alerte agronomique** — farmer has an unacknowledged agronomic alert older than 24h
4. **Pas vu depuis 14+ jours** — `lastVisitDate` is more than 14 days ago AND no visit scheduled
5. Cap at 3 farmers per day (first 3 by priority tier, then by last visit date ascending)

## Implementation

### CF: `getAgentVisitPlan()` → `{ farmers: VisitPlanItem[], generatedAt: Timestamp }`
```typescript
type VisitPlanItem = {
  farmerId: string
  farmerName: string
  province: string
  reasonCode: 'credit_urgent' | 'scheduled' | 'agro_alert' | 'overdue'
  reasonLabel: string   // human-readable French label
  urgencyLevel: 1 | 2 | 3
  lastVisitDate: Timestamp | null
}
```

### New hook: `useAgentVisitPlan` (`src/hooks/useAgentVisitPlan.ts`)
```typescript
// isDevMode() → returns 3 mock visit items
// Real path: httpsCallable(functions, 'getAgentVisitPlan')()
// staleTime: 30 min (plan doesn't change mid-day)
```

### UI changes in `AgentHome.tsx` (both `AgentDesktop` and `AgentMobile`)
- New "Plan du jour" section at top, above the KPI strip
- 3 farmer cards, each showing: name, province, reason badge (color-coded), last visit date
- CTA button on each card: "Commencer la visite →" → navigates to farmer detail + opens report form
- If no visits planned: "Pas de visites urgentes aujourd'hui — Zone à jour ✓" (green)
- Section collapsed by default on mobile (expandable), open by default on desktop

## Acceptance criteria
- [ ] Agent dashboard shows "Plan du jour" section with up to 3 farmers
- [ ] Each visit card shows farmer name, province, reason badge, last visit date
- [ ] Reason badge colors: red (credit urgent), amber (scheduled), gray (overdue)
- [ ] "Commencer la visite" button navigates to the farmer's report form
- [ ] If 0 farmers meet criteria: success state shown ("Zone à jour")
- [ ] Section is collapsed/expandable on mobile
- [ ] `isDevMode()` returns 3 deterministic mock visit items
- [ ] React Query caches the plan for 30 min (no re-fetch on tab switch)

## Smoke test steps
1. Log in as agent → verify "Plan du jour" section appears at top of dashboard
2. Verify 1–3 farmer cards with reason badges (red/amber/gray)
3. Tap "Commencer la visite" → verify navigation to farmer detail / report form
4. If no urgent farmers: verify "Zone à jour" success state
5. Mobile viewport (375px): verify section is collapsed by default, expands on tap
6. Switch tabs and return → verify no loading spinner (cached data)
