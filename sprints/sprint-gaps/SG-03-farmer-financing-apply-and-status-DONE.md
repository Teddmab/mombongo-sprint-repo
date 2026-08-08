# SG-03 — Farmer: Apply for Financing + Real Status View

## Why this matters
The Financement screen exists for farmers but shows mock data. A farmer cannot submit a real financing request from the app, and they cannot see the real status of their application. This is the core value proposition for farmers.

## Current state
- `AgricultorFinancement.tsx` — disbursement history, progress bar, agent card all from `mock.ts`
- "Demander un financement" button on farmer home navigates to the investor Financement screen (wrong!)
- No `FinancingApplicationModal` for farmers
- `getMyFinancingApplications` CF is specified in S4-02 but not yet implemented
- `submitFinancingApplication` CF does not exist

## Work items

### 1. Financing application form (farmer)
Simple 4-step wizard, one step per screen (not a long scrolling form):

**Step 1 — Quel projet ?**
- Crop type (from farmer's parcels if available, else dropdown)
- Surface to finance (ha)
- Purpose: semences / engrais / équipement / irrigation / autre

**Step 2 — Combien ?**
- Amount needed in USD (with a suggested range based on surface × crop type)
- Show equivalence in CDF dynamically
- Duration: 3 / 6 / 9 / 12 months (large tap targets, not a text input)

**Step 3 — Votre situation**
- Do you already have an agent assigned? (yes/no — if yes, auto-fills from `users/{uid}.agentId`)
- Province of the field (if not already in profile)
- Brief description of the project (optional, 2 lines max)

**Step 4 — Confirmer**
- Summary card of the request
- "Soumettre ma demande" button → calls `submitFinancingApplication` CF
- CF creates doc in `financing_applications` with `status: 'pending'`, `farmerId: uid`

### 2. `submitFinancingApplication` CF
```typescript
// financing_applications/{id}
{
  farmerId: uid,
  farmerName: displayName,
  cropType, surfaceHa, purpose,
  requestedAmountUsd, durationMonths,
  province, description,
  status: 'pending',       // pending → approved → disbursing → completed
  agentId: agentId ?? null,
  tranches: [],
  disbursedAmountUsd: 0,
  createdAt: serverTimestamp(),
}
```

### 3. `getMyFinancingApplications` CF
- Returns all `financing_applications` docs where `farmerId == uid`
- Ordered by `createdAt desc`

### 4. Wire `AgricultorFinancement.tsx` to real data
- Replace mock data with `useQuery` → `getMyFinancingApplications`
- Show status badge per application (pending / approved / disbursing / completed / rejected)
- Show disbursement tranches for approved applications
- Show assigned agent's name and contact

### 5. "Can I apply for more?" logic
- If the farmer has an active application (status not `completed` or `rejected`), disable the apply button and show: "Vous avez déjà une demande en cours"
- If no active application, show the apply button prominently

### 6. Simple status timeline
- For each application, show a simple 4-step progress bar:
  - Demande reçue → En étude → Approuvée → Financement en cours → Remboursement
- No numbers, no jargon — big colored dots with simple labels

## Acceptance criteria
- [ ] Farmer can submit a financing application from the app (calls real CF)
- [ ] Submitted application appears immediately in their Financement screen
- [ ] Status updates from admin reflect in real time (next app open / refresh)
- [ ] Farmer with active application cannot submit a duplicate
- [ ] Simple 4-step progress bar shows where their application is
