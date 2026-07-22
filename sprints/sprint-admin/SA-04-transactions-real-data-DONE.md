# SA-04 — Transactions: Real Data + Actions

## Goal
Replace mock `adminTx` array with live `transactions` Firestore collection. Like AdminUsers, a correctly-implemented `src/pages/AdminTransactions.tsx` already exists with a real query — it's just shadowed by the mock in `Admin.tsx`.

## Current state
- `Admin.tsx` exports mock `AdminTransactions` and `AdminTransactionDetail` using `adminTx` from `adminData.ts`
- `src/pages/AdminTransactions.tsx` (separate file) is wired to `adminService.getTransactions()` — NEVER imported by `App.tsx`
- "CSV" download button has no handler
- No approve/reject/refund actions in TransactionDetail

## Work items

### 1. Swap import in App.tsx
- Same pattern as SA-02: import from the real `src/pages/AdminTransactions.tsx`
- Delete mock version from `Admin.tsx`

### 2. Wire TransactionDetail
- Fetch transaction by ID from `transactions/{id}`
- Show full timeline: created → processing → completed/failed
- Fetch related user name from `users/{userId}`

### 3. Approve / Reject actions (for pending transactions)
- `updateDoc(transactions/{id}, { status: 'completed' | 'rejected', resolvedAt: serverTimestamp() })`
- Only show action buttons when status is `pending` or `processing`

### 4. CSV export
- Client-side: convert current filtered results to CSV and trigger download
- Columns: id, type, amount, currency, userId, status, createdAt

## Acceptance criteria
- [ ] Transactions table shows real data from `transactions` collection
- [ ] Kind and search filters work
- [ ] TransactionDetail shows real transaction with timeline
- [ ] Approve/Reject writes status to Firestore
- [ ] CSV download works
