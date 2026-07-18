# SA-01 — Dashboard Real Data

## Goal
Replace all hardcoded strings and mock arrays on AdminDashboard with live Firestore data. `adminService.getDashboardKpis()` is already implemented — it just needs to be wired in.

## Current state
- All 4 KPI tiles show hardcoded strings ("1,284", "$248,200", "186", "96.4%")
- Monthly chart uses inline hardcoded `monthlyData` array
- Role distribution pie chart uses inline hardcoded `distData`
- Recent activity table uses `adminTx` mock array
- Alerts widget uses `adminAlerts` mock array
- `adminService.getDashboardKpis()` reads `users`, `investments`, `farmer_financing` — never called

## Work items

### 1. Wire KPI tiles
- Call `adminService.getDashboardKpis()` via `useQuery`
- Map results to the 4 KPI cards: active users, total invested (USD), financed farmers, avg KYC approval rate
- Show loading skeletons while fetching

### 2. Fix activity chart
- `adminService.getActivity()` is permanently stubbed (hardcoded Mon-Fri array)
- Replace with real query: last 30 days of `transactions` grouped by day (use `orderBy('createdAt') + limit(200)` then aggregate client-side)
- Or simplify: last 7 days of `users` createdAt count per day

### 3. Recent activity table
- Replace `adminTx` mock with last 10 docs from `transactions` collection (orderBy createdAt desc)
- Show: type, amount, user, status, date

### 4. Alerts widget
- Replace `adminAlerts` mock with docs from `admin_alerts` collection (create if needed)
- Or derive from Firestore: pending KYC count, pending financing applications count, unresolved reports count

## Acceptance criteria
- [ ] KPI tiles show real counts from Firestore on page load
- [ ] Loading skeletons shown during fetch
- [ ] Recent transactions table shows last 10 real transactions
- [ ] No hardcoded numbers remain in the component
