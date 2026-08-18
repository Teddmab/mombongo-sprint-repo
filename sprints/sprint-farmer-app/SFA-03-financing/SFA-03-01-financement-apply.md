# SFA-03-01 — Financing Apply + Status Tracker (Farmer App)

## Context
Web sprint SG-03 fully implements the farmer financing flow (apply, status tracker, disbursement). Mobile `FarmerFinancementScreen.tsx` exists but is not wired to real CFs. This sprint wires it.

CFs already deployed:
- `applyForFinancing` — submits application with exploitation + crop details
- `getFinancingStatus` — returns current application status + timeline
- `getMyFinancingApplications` — list of farmer's applications

## Scope
- Wire `FarmerFinancementScreen.tsx` to `getMyFinancingApplications` CF
- Create `ApplyForFinancingSheet.tsx` — multi-step application form
- Create `FinancingStatusCard.tsx` — status timeline (submitted → under review → approved → disbursed)
- Create `src/hooks/useMyFinancing.ts`

## Files to create / modify
- `src/hooks/useMyFinancing.ts`
- `src/screens/financement/FarmerFinancementScreen.tsx` — wire
- `src/screens/financement/ApplyForFinancingSheet.tsx`
- `src/components/FinancingStatusCard.tsx`

## Implementation

### `src/hooks/useMyFinancing.ts`
```typescript
export function useMyFinancing() {
  return useQuery({
    queryKey: ['myFinancing'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_FINANCING_APPLICATIONS
      const res = await httpsCallable<void, { applications: FinancingApplication[] }>(
        functions, 'getMyFinancingApplications'
      )()
      return res.data.applications
    },
  })
}

export function useApplyForFinancing() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (payload: FinancingApplicationPayload) =>
      httpsCallable(functions, 'applyForFinancing')(payload),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['myFinancing'] }),
  })
}
```

### FinancingStatusCard.tsx
```typescript
const STATUS_STEPS = ['Soumis', 'En révision', 'Approuvé', 'Décaissé']
// Vertical timeline with step indicators and dates
// Current step highlighted
// "Votre dossier est traité sous 5–7 jours ouvrables" message when in review
```

### ApplyForFinancingSheet.tsx
```typescript
// Step 1: Select exploitation + crop
// Step 2: Amount requested (CDF) + purpose
// Step 3: Preferred repayment period (months)
// Step 4: Review + submit
```

## Acceptance criteria
- [ ] Financement tab shows list of applications from CF
- [ ] Status timeline shows correct step for each application
- [ ] "Faire une demande" button opens multi-step sheet
- [ ] Sheet submits to `applyForFinancing` CF
- [ ] After submission, new application appears in list with status "Soumis"
- [ ] Dev mode shows mock applications

## Smoke test
1. Open Financement tab — confirm applications list loads
2. Verify status timeline is correct for a "under review" application
3. Tap "Faire une demande" → complete 4-step form → submit
4. Confirm new application appears with "Soumis" status
5. Firebase console → verify application document created
