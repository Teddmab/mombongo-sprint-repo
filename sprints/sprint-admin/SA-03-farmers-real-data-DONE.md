# SA-03 — Farmers: Real Data + Approve/Reject

## Goal
Replace mock `adminFarmers` array with live `farmers` Firestore collection. `adminService.getFarmers()` and `adminService.approveFarmer()` are already implemented — they just need to be wired in.

## Current state
- AdminFarmers uses `adminFarmers` from `adminData.ts` (6 hardcoded objects)
- AdminFarmerDetail looks up by id in the same mock array
- `adminService.getFarmers(filters?)` — real Firestore query, never called
- `adminService.approveFarmer(id)` — real `updateDoc`, never called
- "Vérifier" (approve) button in FarmerDetail has no onClick handler
- "Rapports terrain" tab: 3 hardcoded report entries
- "Documents" tab: 3 hardcoded document names

## Work items

### 1. Wire AdminFarmers list
- Use `adminService.getFarmers()` via `useQuery`
- Keep existing filter UI (status, search) — filter against real results
- Loading skeleton while fetching

### 2. Wire AdminFarmerDetail
- Fetch farmer doc by ID from `farmers/{id}`
- Fetch agent reports from `agent_reports` where `farmerId == id` (orderBy visitDate desc)
- Fetch financing applications from `financing_applications` where `farmerId == id`

### 3. Wire approve action
- "Vérifier" button calls `adminService.approveFarmer(id)` → sets `status: 'verified'`
- Confirmation dialog before write
- Invalidate query cache after success

### 4. Add reject action
- `updateDoc(farmers/{id}, { status: 'rejected', rejectionReason: string })`
- Modal with reason text field

### 5. Documents tab
- List from `farmers/{id}/documents` subcollection or `status` field attachments
- "Voir" opens URL

## Acceptance criteria
- [ ] Farmers list shows real data from `farmers` collection
- [ ] Status filter works against real data
- [ ] FarmerDetail shows real farmer + agent reports + financing apps
- [ ] Approve/Reject writes to Firestore and refreshes the page
