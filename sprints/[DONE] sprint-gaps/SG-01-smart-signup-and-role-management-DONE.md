# SG-01 — Smart Signup & Role Management

## Why this matters
Right now every role sees the exact same signup form. A farmer is not a tech-savvy investor. If someone picks the wrong role, they're stuck — there is no self-service fix.

## Already implemented (skip these)
- `quick-fields` step with phone, province, cropType, businessType — ✅ exists in AuthScreen.tsx
- `welcomeStep` rendered after signup — ✅ done
- `OnboardingModal` with role-specific slides (farmer/investor/merchant/agent) — ✅ fully built
- Google sign-in → no profile → redirect to role selection — ✅ already wired (AuthScreen.tsx ~line 207)

## Remaining work items

### 1. Remove agent from `GUIDED_ROLES` (3 choices only)
- `GUIDED_ROLES` array in AuthScreen.tsx still has agent (line 34)
- Remove agent entry — agent is invite-only, not self-selectable
- The existing `ALL_ROLES` array (used for multi-role login picker) keeps agent — only `GUIDED_ROLES` changes

### 2. Add missing role-specific quick-fields
Farmer step is missing:
- Surface approximative: radio/select "< 1 ha / 1–5 ha / > 5 ha" (optional)

Investor step is missing:
- Country of residence: "DRC" | "Diaspora" toggle (optional)
- How they heard: optional text or select

### 3. Role-change request form in Settings/Profile
- On ProfileScreen → Settings row → "Demander un changement de rôle"
- Simple modal: current role (read-only), desired role (select from 3), reason (textarea)
- Calls `createRoleChangeRequest` CF → writes to `role_change_requests/{id}`
- Show confirmation toast; no reload needed
- CF: `createRoleChangeRequest(uid, { currentRole, requestedRole, reason })` → `role_change_requests`

## Acceptance criteria
- [ ] Signup shows 3 choices (investor / farmer / merchant), agent removed
- [ ] Farmer quick-fields include surface option
- [ ] Investor quick-fields include country of residence
- [ ] Role-change request modal accessible from ProfileScreen, writes to Firestore
