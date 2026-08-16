# SA-08 — Real data wiring: all admin pages off mock

**Status:** DONE  
**Branch:** `feature/sa-real-data-all` (mombongo-admin)

## Problem
`mombongo-admin/src/App.tsx` imported all core pages from the monolithic mock `Admin.tsx` (1 730 lines, all hardcoded array data). Standalone real-data files existed for Dashboard, Users, and Transactions stubs, but were not wired in. Farmers, KYC, Alerts, and Settings had no standalone file. AdminAcademia module reorder was stubbed (`/* reorder up/down */`) and moduleCount was never synced.

## Changes

### New standalone pages
| File | Exports | Data source |
|------|---------|-------------|
| `AdminFarmers.tsx` | `AdminFarmers`, `AdminFarmerDetail` | `adminService.getFarmers()`, `adminService.approveFarmer()`, `updateDoc` reject |
| `AdminKyc.tsx` | `AdminKyc` | Firestore `users` collection filtered by `kycStatus`, `updateDoc` validate/reject |
| `AdminAlerts.tsx` | `AdminAlerts` | `getCountFromServer` on users/farmers/transactions/financing, recent-failed feed |
| `AdminSettings.tsx` | `AdminSettings` | Firestore `platform_settings/global` doc via `setDoc` (merge) |

### Expanded stubs → full pages
| File | Changes |
|------|---------|
| `AdminTransactions.tsx` | Added type/status filters, search, CSV export, `AdminTransactionDetail` route page |

### AdminAcademia.tsx fixes
- Added `writeBatch` + `increment` imports
- `saveModule`: after `addDoc`, calls `updateDoc(courseRef, { moduleCount: increment(1) })`
- `deleteModule`: after `deleteDoc`, calls `updateDoc(courseRef, { moduleCount: increment(-1) })`
- `reorderModule(idx, direction)`: uses `writeBatch` to atomically swap `order` field between two adjacent modules

### App.tsx
Replaced single `import { … } from "@/pages/Admin"` with individual imports from each standalone file. `AdminLayout`, `AdminUserDetail`, `AdminOpportunities`, `AdminReports`, `AdminReportDetail` remain from `Admin.tsx` (layout/reports are not mock-data concerns for this sprint).

## Sprint docs renamed to DONE
SA-01 through SA-07 all renamed to `*-DONE.md`.
