# SAG-03-02 — Agent Financing Actions (Agent App)

## Context
From the pipeline kanban, agents need to take actions on financing applications: submit an assessment score, recommend approval/rejection, and confirm disbursement receipt. These actions call CFs already deployed (SG-05 and SG-10).

CFs available:
- `submitAgentAssessment` — input: `{ applicationId, score: 1–5, notes, recommendation: 'approve'|'reject' }` → advances stage from "evaluate" to "reviewing"
- `confirmDisbursementReceipt` — input: `{ applicationId }` → advances from "approved" to "disbursed"
- `updateFinancingStatus` — admin/agent action to update status (already deployed)

## Scope
- Create `AssessmentSheet.tsx` — bottom sheet for submitting assessment from pipeline "À évaluer" cards
- Create `DisbursementConfirmSheet.tsx` — confirm disbursement from "Approuvé" cards
- Wire action buttons in `AgentPipelineScreen.tsx`
- After any action, invalidate pipeline query (card moves to next column)

## Files to create
- `src/screens/pipeline/AssessmentSheet.tsx`
- `src/screens/pipeline/DisbursementConfirmSheet.tsx`

## Implementation

### AssessmentSheet.tsx
```typescript
// Opens from "Faire rapport" (after report submitted, assessment auto-attaches) or standalone
// Fields:
//   Score (1–5 stars with labels: Très risqué → Excellent)
//   Notes (textarea, max 500 chars)
//   Recommendation: Approuver / Rejeter (2 buttons, color-coded)
// Submit → submitAgentAssessment CF → invalidate pipeline
```

### DisbursementConfirmSheet.tsx
```typescript
// Simple confirmation: "Confirmez-vous que Jean Kasongo a reçu 250 000 FC?"
// Amount shown (from pipeline card data)
// Signature or photo of disbursement receipt (optional, using KYC upload pattern)
// Confirm button → confirmDisbursementReceipt CF
```

## Acceptance criteria
- [ ] "Valider" button on "À évaluer" card opens AssessmentSheet
- [ ] Assessment sheet submits score + recommendation to CF
- [ ] After submit, card moves to "En cours d'instruction" column (re-fetch)
- [ ] "Confirmer réception" on "Approuvé" card opens DisbursementConfirmSheet
- [ ] After confirm, card moves to "Décaissé" column
- [ ] Rejection reason must be entered (required field) before submit

## Smoke test
1. Open pipeline — find a farmer in "À évaluer"
2. Tap "Valider" → fill assessment → submit
3. Confirm card moves to "En cours d'instruction"
4. Find farmer in "Approuvé" → tap "Confirmer réception"
5. Confirm card moves to "Décaissé"
