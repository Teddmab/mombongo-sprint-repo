# SG-05 — Agent: Wire Report Submission + Report History

## Why this matters
The agent report form is the primary tool for field agents. It exists and looks good — but it submits to nowhere. No data goes to Firestore. The farmer list is mock. Agents also have no way to look back at their own submitted reports.

## Current state
- `AgentReportScreen.tsx` — submit fires `toast.success` without calling any CF
- Farmer selector uses `agentFarmers` from mock data (`mock.ts`)
- `submitAgentReport` CF is specified in S4-03 but the web submit is not wired
- `getAgentFarmers` CF is specified but not implemented
- No dedicated "mes rapports" screen for agents
- Report history on `AgentHome` and `AgentFinancement` uses `agentReports` mock array

## Work items

### 1. `getAgentFarmers` CF
Returns all `farmers` docs where `agentId == uid`:
```typescript
// Returns: { id, name, crop, region, stage, status, lastVisit, daysToHarvest, surfaceHa }[]
```

### 2. Wire farmer selector in `AgentReportScreen`
- Replace mock `agentFarmers` with `useQuery → getAgentFarmers` CF
- Show loading state while fetching
- If no farmers assigned: show empty state "Aucun agriculteur assigné — contactez votre superviseur"
- Default-select the first farmer from the list

### 3. Wire `submitAgentReport` CF call
The `submit` function in `AgentReportScreen.tsx` already has the real call written but inside an `isDevMode()` branch that never runs in prod. Fix the logic:
```typescript
// Current (wrong): if (isDevMode()) { fake delay } else { real call }
// Fix: always call real CF in prod; fake delay only in dev
```
CF writes a doc to `agent_reports` collection.

### 4. Photo upload in reports
- After photos are selected in the form, call `getAgentReportUploadUrl` CF to get signed URLs
- Upload photos directly to Storage using the signed URLs
- Pass the resulting URLs as `photoUrls[]` to `submitAgentReport`
- Max 3 photos per report; compress to <500KB before upload

### 5. Report history screen (`/rapport/historique`)
- New screen accessible from AgentHome "Mes rapports" quick action
- Lists all `agent_reports` where `agentId == uid` (via `getMyAgentReports` CF)
- Each row: farmer name, visit date, condition badge (good/average/poor), status (submitted/reviewed)
- Tap row → report detail (read-only view of all fields)
- Pull-to-refresh (mobile)

### 6. Wire report history on AgentHome and AgentFinancement
- Replace `agentReports` mock with `useQuery → getMyAgentReports` (last 5)

### 7. GPS check-in (simple version)
- When opening the report form, auto-capture device location via `navigator.geolocation`
- Store as `gpsLat`, `gpsLng` in the `agent_reports` doc
- Show a small "📍 Position enregistrée" badge when captured
- If location denied: silently skip (no blocking error)

## Cloud Functions needed
- `getAgentFarmers(uid)` — farmers where `agentId == uid`
- `submitAgentReport(data)` — creates `agent_reports/{id}` doc (already spec'd in S4-03)
- `getMyAgentReports(uid, limit?)` — reports where `agentId == uid`, orderBy visitDate desc
- `getAgentReportUploadUrl(uid, reportId, fileIndex)` — signed Storage URL

## Acceptance criteria
- [ ] Farmer list in report form loads from real Firestore via CF
- [ ] Submitting a report creates a real `agent_reports` doc
- [ ] Photos upload via signed URL (no direct Storage SDK)
- [ ] GPS coordinates captured silently when permission granted
- [ ] Report history screen shows all past reports for the agent
- [ ] AgentHome shows last 5 real reports (not mock)
