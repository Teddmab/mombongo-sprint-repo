# SA-06 — Alerts & Settings: Real Data

## Goal
Wire AdminAlerts and AdminSettings to Firestore instead of mock arrays.

## Current state

### AdminAlerts
- Uses `adminAlerts` from `adminData.ts` (4 hardcoded objects)
- No dismiss/read actions, no create

### AdminSettings
- 6 hardcoded setting cards (platform fee, exchange rate, min investment, KYC duration, languages, maintenance mode)
- Each "Modifier" button has no onClick handler

## Work items

### Alerts

#### 1. Wire alert list
- Create `admin_alerts` Firestore collection (or derive alerts from existing collections)
- Derived alerts approach (simpler, no extra collection):
  - Count of `users` where `kycStatus == 'pending'` → "X users awaiting KYC"
  - Count of `financing_applications` where `status == 'pending'` → "X financing applications pending"
  - Count of `agent_reports` submitted in last 7 days → "X new field reports"
- Show severity badge (info / warning / critical)

#### 2. Dismiss action
- If using `admin_alerts` collection: `updateDoc({dismissed: true})`
- If derived: no dismiss needed (auto-resolves when underlying data changes)

### Settings

#### 1. Create `platform_settings` Firestore collection
- Single doc `platform_settings/global` with fields:
  - `platformFeePercent`: number
  - `exchangeRateCdfUsd`: number
  - `minInvestmentUsd`: number
  - `kycApprovalHours`: number
  - `maintenanceMode`: boolean

#### 2. Wire settings read
- `getDoc(platform_settings/global)` on mount
- Display current values in each card

#### 3. Wire settings update
- Each "Modifier" opens inline edit or modal
- `updateDoc(platform_settings/global, { [field]: newValue })`
- Toast on success

## Acceptance criteria
- [ ] Alerts show real derived counts from Firestore
- [ ] Settings show real values from `platform_settings/global`
- [ ] Each setting can be edited and the update persists in Firestore
