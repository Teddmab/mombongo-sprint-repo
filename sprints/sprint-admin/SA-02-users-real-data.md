# SA-02 — Users: Real Data + Actions

## Goal
Replace the mock `adminUsers` array in AdminUsers and AdminUserDetail with live `users` Firestore data. Two correctly-implemented files (`src/pages/AdminUsers.tsx`, `src/pages/AdminTransactions.tsx`) already exist with real queries but are shadowed by the mock implementations in `Admin.tsx`.

## Current state
- `Admin.tsx` exports `AdminUsers` and `AdminUserDetail` using hardcoded `adminUsers` from `adminData.ts`
- `src/pages/AdminUsers.tsx` (separate file) is wired to `adminService.getUsers()` → real `users` collection — but is NEVER imported by `App.tsx`
- "Inviter" button has no handler
- "Éditer" and "Suspendre" buttons in UserDetail have no handlers
- KYC documents, investments, notes tabs are all hardcoded placeholders

## Work items

### 1. Swap import in App.tsx
- Change `import { ..., AdminUsers, AdminUserDetail, ... } from "@/pages/Admin"` to import from the real `src/pages/AdminUsers.tsx`
- Delete or empty the mock versions from `Admin.tsx`

### 2. Wire AdminUserDetail
- Fetch user doc by ID from `users/{id}`
- Fetch user's transactions from `transactions` where `userId == id`
- Fetch user's investments from `investments` where `userId == id` (or via CF)
- Show real KYC status and level

### 3. Suspend / Reactivate action
- `updateDoc(users/{id}, { status: 'suspended' })` with confirm dialog
- Toggle button label based on current status

### 4. Invite user
- Create doc in `users` collection with role + email (or call a CF to send invite email)

### 5. KYC documents tab
- List docs from `users/{id}/kyc_documents` subcollection or a `kyc_documents` collection filtered by userId
- "Voir" button opens document URL

## Acceptance criteria
- [ ] Users table shows real data from `users` collection
- [ ] Search filters correctly against Firestore results
- [ ] UserDetail shows real user data, transactions, investments
- [ ] Suspend/Reactivate writes to Firestore and reflects in the table
