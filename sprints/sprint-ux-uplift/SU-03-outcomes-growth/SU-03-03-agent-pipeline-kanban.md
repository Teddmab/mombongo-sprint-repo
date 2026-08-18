# SU-03-03 — Agent credit pipeline kanban view

**Sprint:** SU-03 · Outcomes & growth  
**Branch:** `feature/su-03-outcomes-growth`  
**Effort:** ~4 days

## Context
The agent manages loan applications across their zone but currently has no overview of where each application stands. This story adds a pipeline view (kanban-style columns on desktop, horizontal scroll on mobile) showing all financing applications grouped by stage.

## Pipeline stages (4 columns)

| Stage | Status values | Label |
|-------|--------------|-------|
| Prospect | `draft`, `not_started` | En prospection |
| Instruction | `submitted`, `under_review`, `documents_requested` | En instruction |
| Approuvé | `approved` | Approuvé |
| Décaissé | `disbursed`, `completed` | Décaissé |

## Implementation

### CF: `getAgentFinancingPipeline()` → `{ columns: PipelineColumn[], totalDisbursedCdf: number }`
```typescript
type PipelineColumn = {
  stage: 'prospect' | 'instruction' | 'approved' | 'disbursed'
  count: number
  totalAmountCdf: number
  items: Array<{ farmerId, farmerName, amountCdf, daysSinceLastUpdate, urgency }>
}
```

### New hook: `useAgentPipeline` (`src/hooks/useAgentPipeline.ts`)
- `isDevMode()` → mock pipeline: Prospect 5, Instruction 3 (1 urgent), Approuvé 2, Décaissé 8

### UI: New "Pipeline crédit" section on agent dashboard

**Desktop:** 4-column kanban layout
- Column header: stage name + count badge + total CDF
- Each card: farmer name, amount, days since update, urgency indicator
- Urgent cards (> 5 days in Instruction with no action): red border

**Mobile:** Horizontal scrollable columns (swipe to see next stage)
- Same card format, compact

Added to agent dashboard below the Visit Plan section (SU-02-02), above the farmer list.

Tab navigation in agent dashboard:
- **Résumé** (existing KPI strip)
- **Plan du jour** (SU-02-02)
- **Pipeline** (new, this story)
- **Agriculteurs** (existing farmer list)

## Acceptance criteria
- [ ] Pipeline section renders on agent dashboard (desktop: 4 columns, mobile: swipeable)
- [ ] Column counts match actual financing application statuses for this agent
- [ ] Cards > 5 days in Instruction show red urgency border
- [ ] Tapping a card navigates to the farmer's detail screen
- [ ] Total disbursed CDF shown in dashboard KPI strip
- [ ] `isDevMode()` returns mock pipeline with 18 total applications
- [ ] Empty column shows "Aucun dossier" (not blank space)

## Smoke test steps
1. Log in as agent → navigate to "Pipeline" tab → verify 4 columns appear (desktop)
2. Verify application counts match mock data (5 / 3 / 2 / 8)
3. Verify a card with > 5 days in Instruction shows red border
4. Tap a farmer card → verify navigation to farmer detail
5. Mobile (375px) → verify columns are horizontally scrollable, no overflow
6. Empty column scenario (set isDevMode mock to return empty Approuvé column) → verify "Aucun dossier"
