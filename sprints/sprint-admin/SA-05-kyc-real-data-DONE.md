# SA-05 — KYC: Real Data + Validate/Reject

## Goal
Replace mock KYC list with real `users` collection filtered by pending KYC status. Wire Validate and Reject buttons to Firestore updates.

## Current state
- AdminKyc filters `adminUsers` where `kycLevel < 3` — produces 4 mock entries
- "Valider" and "Rejeter" buttons render but have no onClick handlers
- No document preview functionality

## Work items

### 1. Wire KYC list
- Query `users` collection where `kycStatus == 'pending'` (or `kycLevel < 3`)
- Display: name, email, submission date, document types submitted, current level
- Loading skeleton

### 2. Validate action
- `updateDoc(users/{id}, { kycStatus: 'approved', kycLevel: 3, kycApprovedAt: serverTimestamp() })`
- Confirm dialog
- Invalidate query on success

### 3. Reject action
- Modal with rejection reason field
- `updateDoc(users/{id}, { kycStatus: 'rejected', kycRejectionReason: string, kycRejectedAt: serverTimestamp() })`

### 4. Document preview
- Fetch document URLs from `users/{id}/kyc_documents` subcollection or URL fields on the user doc
- Open in new tab or lightbox

### 5. KYC stats banner
- Show counts: pending / approved today / rejected this week

## Acceptance criteria
- [ ] KYC list shows real pending users from Firestore
- [ ] Validate writes kycStatus: approved and refreshes list
- [ ] Reject with reason writes kycStatus: rejected and refreshes list
- [ ] Documents are viewable
