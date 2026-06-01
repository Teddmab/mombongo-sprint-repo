# SP-01 — Role-Based Screens & Profile Plans

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-01 |
| Sprint | Patch — Point-in-time corrections & quick implementations |
| Branch | `feature/s1-03-home-screen` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Date | 2026-06-01 |
| Type | Quick implementation |

## Scope — `mombongo-web`

### What was done

Complete role-based UI differentiation across all main screens. Previously, `"farmer"` and `"agent"` roles shared the investor interface. All screens now render a distinct, role-appropriate view.

---

## Changes

### 1. Home Screen — `HomeScreen.tsx`

Replaced the placeholder `FarmerHome` stub with two fully-built role homes.

**Files created:**
- `src/pages/home/AgricultorHome.tsx` — `AgricultorDesktop` + `AgricultorMobile`
- `src/pages/home/AgentHome.tsx` — `AgentDesktop` + `AgentMobile`

**Agricultor home shows:**
- KPI strip: financing received (%), days to harvest, current growth stage, surface (ha)
- 6-step crop cycle timeline (Semis → Récolte) with active stage highlighted
- Financing progress bar
- Cultural calendar task list with type icons (irrigation, traitement, récolte, visite)
- Alert feed — urgent alerts in red, weather/market/agent/payment
- Agent card (name + next visit date)

**Agent home shows:**
- KPI strip: farmers assigned, reports this month, urgent visits, next visit
- Farmer list sorted by urgency (urgent → attention → ok) with status badges
- Recent reports with validated/pending/rejected status
- Status chips on mobile

---

### 2. Market Screen — `MarketScreen.tsx`

**Files created:**
- `src/pages/market/AgricultorMarket.tsx`
- `src/pages/market/AgentMarket.tsx`

**Agricultor view:** "Mes annonces" — publish crops, track listing status and funding %, tab to see market prices.

**Agent view:** Zone farmer list with priority sorting — publish products on behalf of farmers, search, inline status badges.

---

### 3. Bourse Screen — `BourseScreen.tsx`

**Files created:**
- `src/pages/bourse/AgricultorBourse.tsx`
- `src/pages/bourse/AgentBourse.tsx`

**Agricultor view:** Price tracker filtered to their own crops + "Mettre en vente" CTA. Opportunity alert banner when a crop price rises.

**Agent view:** Full price feed with farmer names linked to their crop symbol. "Alerter" button appears when a farmer's crop is up — lets agent notify them to sell.

---

### 4. Financement Screen — `FinancementScreen.tsx`

**Files created:**
- `src/pages/financement/AgricultorFinancement.tsx`
- `src/pages/financement/AgentFinancement.tsx`

**Agricultor view:** Their own financing request — disbursement history, progress bar, agent card, upcoming cultural tasks.

**Agent view:** Full farmer management — filter by urgent/attention/ok, inline Rapport button per farmer, recent report history with status.

---

### 5. Profile Screen — `ProfileScreen.tsx`

**Per-role subscription plans** replacing the hardcoded `PLANS[1]` (investisseur) for all roles.

| Role | Plan | Price |
|------|------|-------|
| investor | Investisseur | $9.99/mo |
| farmer | Agriculteur Essentiel | $4.99/mo |
| agent | Agent Pro | $7.99/mo |
| merchant | Premium | $24.99/mo |

Desktop and mobile subscription cards now show plan features inline, correct badge, and role-appropriate CTA.

**Per-role wallet section:**
- Investor/Merchant: full balance + Dépôt/Retrait buttons
- Farmer: financing received ($650) with 65% progress bar, no payment actions
- Agent: monthly commissions display, no payment actions

---

### 6. Navigation — role badge

**Files modified:** `src/components/BottomNav.tsx`, `src/components/AppShell.tsx`

Role badge added to both nav bars:
- **Desktop** (TopNav): colored pill below display name, top-right
- **Mobile** (MobileNavBar): inline badge next to page title

| Role | Color |
|------|-------|
| Investisseur | Blue |
| Agriculteur | Green |
| Agent terrain | Amber |
| Commerçant | Purple |

---

### 7. Mock data — `src/data/mock.ts`

New exports added (all mock, no Firestore):
- `cropTasks` — cultural calendar tasks with types and done/pending state
- `farmerAlerts` — weather/market/agent/payment alerts with urgent flag
- `agentFarmers` — 6 farmer cards with status, crop, region, harvest countdown
- `agentReports` — 5 submitted reports with validated/pending/rejected status
- `myListings` — 3 agricultor product listings with funding progress

---

## Notes

- All new role views use **mock data only** — no Firestore hooks added
- Investor experience is **unchanged** — all dispatch logic preserves existing behavior
- To switch roles for testing: `localStorage.setItem("mb_role", "farmer")` then reload
- `SubscriptionModal` still opens for all roles but shows investor-oriented plans — role-specific subscription flows are a future sprint

---

## Sprint Patch — Naming Convention

This folder (`sprint-patch/`) is reserved for:
- **Quick implementations** that don't belong to a numbered sprint but are ready to ship
- **Point-in-time corrections** — UI fixes, data wiring, role differentiation
- **Cross-sprint improvements** that affect multiple existing screens

Files in this folder follow `SP-NN-slug.md` naming where N is a sequential patch number.
