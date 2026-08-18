# SAG-01-04 — Profile Management (Agent App)

## Context
Agents need to manage their own profile: display name, phone number, avatar, and language preference. Unlike farmers, agents do not have a wallet, subscription, or KYC flow — their account is created by the Mombongo admin team. Password change is still available via Firebase Auth client SDK.

The `updateProfile` CF (already deployed, shared with farmer app) handles name + phone updates. Avatar upload uses the same signed URL pattern as SFA-00-05.

## Scope
- Wire `ProfileScreen.tsx` (agent variant) to `updateProfile` CF
- Edit: display name, phone, province assignment (read-only — set by admin)
- Avatar upload via camera or gallery
- Language picker (FR / SW / LN) — saves to AsyncStorage + i18n
- Change password via Firebase Auth client SDK
- Show agent's assigned zone/province (read-only from `userProfile`)
- Show today's visit count (from `useAgentVisitPlan`)
- No wallet, no subscription, no KYC sections

## Files to modify
- `src/screens/ProfileScreen.tsx` — add `isAgentApp` guard to hide wallet/subscription rows and add agent-specific fields

## Implementation

### ProfileScreen.tsx — agent-specific sections
```typescript
import { isAgentApp } from '@/constants/appRole'

// Shown for agent only — zone is set by admin, read-only:
{isAgentApp && (
  <ReadOnlyRow label="Zone assignée" value={userProfile?.province ?? '—'} />
)}

// Shown for agent only — today's visit count:
{isAgentApp && (
  <StatRow label="Visites aujourd'hui" value={visitPlan?.visits.length ?? 0} />
)}

// Hidden for agent — no wallet:
{!isAgentApp && <WalletBalanceRow />}

// Hidden for agent — no subscription:
{!isAgentApp && <SubscriptionRow />}

// Hidden for agent — no KYC:
{!isAgentApp && <KycStatusRow />}
```

### Shared profile hooks (reuse SFA-00-05)
```typescript
// useUpdateProfile, useChangePassword — same hooks, no changes needed
// Avatar upload — same useKycUpload pattern with type: 'avatar'
```

### Language selection — same as SFA-00-05
```typescript
const LANGUAGES = [
  { code: 'fr', label: 'Français' },
  { code: 'sw', label: 'Kiswahili' },
  { code: 'ln', label: 'Lingala' },
]
```

## Acceptance criteria
- [ ] Agent profile screen shows name, phone, avatar, language picker
- [ ] "Zone assignée" shown as read-only (set by admin)
- [ ] Today's visit count shown from visit plan data
- [ ] No wallet, subscription, or KYC rows visible
- [ ] Name/phone edits call `updateProfile` CF
- [ ] Avatar upload works via signed URL
- [ ] Language change persists across restarts
- [ ] Change password works via Firebase Auth client SDK

## Smoke test
1. Open Profile in agent build → confirm no wallet/KYC/subscription rows
2. Confirm "Zone assignée" shows agent's province (read-only)
3. Edit display name → save → verify change persists
4. Tap avatar → pick photo → verify avatar updates
5. Change language to Lingala → kill + reopen → Lingala persisted
