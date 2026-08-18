# SAG-03-01 — Financing Pipeline Kanban (Agent App)

## Context
Port of web sprint SU-03-03. The agent manages a portfolio of farmers at different stages in the financing process. The kanban gives an at-a-glance view: which farmers need the agent's action vs. which are waiting on Mombongo HQ.

4 columns (same as web):
1. **À évaluer** — application submitted, agent must do field assessment
2. **En cours d'instruction** — agent completed assessment, HQ reviewing
3. **Approuvé** — financing approved, agent coordinates disbursement
4. **Décaissé** — funds released, agent monitors first repayment

The `getAgentFinancingPipeline` CF (planned in SU-03-03) returns farmers grouped by column.

## Scope
- Create `src/hooks/useAgentPipeline.ts` — calls `getAgentFinancingPipeline` CF
- Create `src/screens/pipeline/AgentPipelineScreen.tsx` — horizontal column layout
- Each farmer card: name, amount, days in current stage, action button
- Action buttons per column: "Faire rapport" (col 1), "Valider rapport" (col 2), "Confirmer réception" (col 3)
- Tapping a card → AgentFarmerDetailScreen

## Cloud Function required
`getAgentFinancingPipeline` — planned in SU-03-03. Returns:
```typescript
{
  columns: {
    evaluate: FinancingCard[]
    reviewing: FinancingCard[]
    approved: FinancingCard[]
    disbursed: FinancingCard[]
  }
  totalPendingActions: number
}
```

## Files to create
- `src/hooks/useAgentPipeline.ts`
- `src/screens/pipeline/AgentPipelineScreen.tsx`

## Implementation

### `src/hooks/useAgentPipeline.ts`
```typescript
const MOCK_PIPELINE = {
  columns: {
    evaluate: [
      { id: 'f1', farmerName: 'Jean Kasongo', amountCdf: 250_000, daysInStage: 2 },
    ],
    reviewing: [
      { id: 'f2', farmerName: 'Marie Lwamba', amountCdf: 180_000, daysInStage: 7 },
    ],
    approved: [],
    disbursed: [
      { id: 'f3', farmerName: 'Pierre Mwamba', amountCdf: 120_000, daysInStage: 14 },
    ],
  },
  totalPendingActions: 1,
}

export function useAgentPipeline() {
  return useQuery({
    queryKey: ['agentPipeline'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_PIPELINE
      const res = await httpsCallable<void, typeof MOCK_PIPELINE>(
        functions, 'getAgentFinancingPipeline'
      )()
      return res.data
    },
    staleTime: 5 * 60_000,
    refetchInterval: 5 * 60_000,
  })
}
```

### AgentPipelineScreen.tsx — horizontal scroll columns
```typescript
// ScrollView horizontal: each column is full-height, ~300px wide
// Column header: title + count badge (e.g., "À évaluer 2")
// "Action requise" banner on "À évaluer" column when count > 0
// Each card:
//   Farmer name + amount in CDF (formatted)
//   "X jours dans cette étape" — red if > threshold
//   Action button specific to column
// Pull to refresh
```

## Acceptance criteria
- [ ] Pipeline tab shows 4 columns with correct counts
- [ ] "À évaluer" column shows farmers needing agent field assessment
- [ ] "Faire rapport" button pre-fills report form with correct farmer
- [ ] Days in stage shown; red when overdue (> 5 days in "À évaluer", > 10 in others)
- [ ] Auto-refreshes every 5 min
- [ ] Badge on Pipeline tab shows `totalPendingActions` count

## Smoke test
1. Open Pipeline tab — confirm 4 columns load
2. Verify mock farmer appears in "À évaluer" column
3. Tap "Faire rapport" — confirm report form opens with correct farmer
4. Pull to refresh — no crash
5. Tab badge shows 1 (from totalPendingActions: 1)
