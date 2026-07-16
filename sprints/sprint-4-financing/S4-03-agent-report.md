# S4-03 — Financing — Agent Field Report

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S4-03 |
| Sprint | Sprint 4 — Financing |
| Branch | `feature/s4-03-agent-report` |
| Merges into | `dev` |
| Estimate | 1h (UI done — backend only) |
| Dependencies | S4-01, S4-00 (farmers collection seeded) |

## Status: UI complete via SP-02 — backend wiring remains

`AgentReportScreen.tsx` was fully built in **SP-02** (601 lines). It is a role-dispatched screen at `/report/new`:

| Role | View |
|------|------|
| Agent | "Rapport de visite terrain" — 6-section form: farmer selector (from mock), visit date, condition scale 1–5, problems checkboxes, financing fields, recommendations + next visit |
| Agriculteur | "Signalement culture" — 4-section form: crop selector, condition scale, urgency checkboxes, message to agent |
| Investor / Merchant | Access-denied screen |

Desktop shows a sticky live-summary sidebar. Form uses spring animations + 800 ms fake-submit loading state + `toast.success`.

**What's still mock:** the farmer selector in the Agent form pulls from `agentFarmers` in `src/data/mock.ts`. The submit handler shows a success toast but does not call a Cloud Function.

---

## Remaining work

### Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Replace mock farmer list with Firestore query; wire submit to `submitAgentReport` |
| `mombongo-functions` | 🔨 Active | `submitAgentReport` onCall |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### submitAgentReport onCall

Create `src/financing/submitAgentReport.ts`:

```typescript
export const submitAgentReport = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const {
    farmerId,
    visitDate,
    cropCondition,      // 1–5 scale
    growthStage,
    surfaceHa,
    problems,           // string[]
    disbursedUsd,
    additionalNeedUsd,
    recommendations,
    nextVisitDate,
    photoUrls,          // uploaded to Storage before call
  } = data

  if (!farmerId || !recommendations)
    throw new functions.https.HttpsError('invalid-argument', 'farmerId and recommendations required')

  const farmerSnap = await db.collection('farmers').doc(farmerId).get()
  if (!farmerSnap.exists)
    throw new functions.https.HttpsError('not-found', 'Farmer not found')

  const now = admin.firestore.FieldValue.serverTimestamp()
  await db.collection('agent_reports').add({
    agentId: uid,
    farmerId,
    visitDate: visitDate ?? now,
    cropCondition,
    growthStage,
    surfaceHa,
    problems: problems ?? [],
    disbursedUsd,
    additionalNeedUsd,
    recommendations,
    nextVisitDate,
    photoUrls: photoUrls ?? [],
    status: 'submitted',
    createdAt: now,
  })

  await db.collection('farmers').doc(farmerId).update({ lastReportAt: now })

  return { success: true }
})
```

**Storage rules** — add to `storage.rules`:
```
match /agent_reports/{agentId}/{filename} {
  allow read:  if request.auth != null;
  allow write: if request.auth != null && request.auth.uid == agentId
    && request.resource.size < 5 * 1024 * 1024
    && request.resource.contentType.matches('image/.*');
}
```

Export in `src/index.ts`.

---

## mombongo-web

### Step 1 — Replace mock farmer list with Firestore

In `AgentReportScreen.tsx`, the Agent form farmer selector currently reads from `agentFarmers` in mock.

Replace with a `useAgentFarmers()` hook (create in `src/hooks/useFinancing.ts`):

```typescript
// No Firestore SDK — all reads through Cloud Functions (db not exported from firebase.ts)
export function useAgentFarmers(agentId: string) {
  return useQuery({
    queryKey: ['agent-farmers', agentId],
    queryFn: async () => {
      if (isDevMode()) return agentFarmers  // keep mock fallback
      const result = await httpsCallable<{ agentId: string }, { farmers: AgentFarmerCard[] }>(
        functions, 'getAgentFarmers'
      )({ agentId })
      return result.data.farmers
    },
    enabled: !!agentId,
  })
}
```

> **`mombongo-functions` dependency**: add `getAgentFarmers` onCall — queries `farmers` where `agentId == uid` ordered by `status`.

### Step 2 — Wire submit to Cloud Function

⚠️ **Architecture rule**: `db` and `storage` are NOT exported from `src/lib/firebase.ts` in `mombongo-web`. The frontend must never call `storageRef`, `uploadBytes`, or `getDownloadURL` directly.

**Photo upload pattern** — use a signed URL from a Cloud Function:

1. Add `getAgentReportUploadUrl` onCall to functions:
```typescript
export const getAgentReportUploadUrl = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')
  const { filename, contentType } = data as { filename: string; contentType: string }
  const path = `agent_reports/${uid}/${Date.now()}-${filename}`
  const [url] = await admin.storage().bucket().file(path).getSignedUrl({
    action: 'write',
    expires: Date.now() + 5 * 60 * 1000,
    contentType,
  })
  return { uploadUrl: url, path }
})
```

2. Frontend upload flow (no Storage SDK import needed):
```typescript
async function uploadPhoto(file: File): Promise<string> {
  const { data } = await httpsCallable<object, { uploadUrl: string; path: string }>(
    functions, 'getAgentReportUploadUrl'
  )({ filename: file.name, contentType: file.type })
  await fetch(data.uploadUrl, { method: 'PUT', body: file, headers: { 'Content-Type': file.type } })
  return data.path  // store the path; admin/functions resolves to URL
}

async function handleSubmit() {
  setSubmitting(true)
  try {
    // 1. Upload photos via signed URL (no direct Storage SDK)
    const photoPaths = await Promise.all(photos.map(uploadPhoto))
    // 2. Submit report
    await httpsCallable(functions, 'submitAgentReport')({
      farmerId, visitDate, cropCondition, growthStage, surfaceHa,
      problems, disbursedUsd, additionalNeedUsd, recommendations,
      nextVisitDate, photoPaths,
    })
    toast.success(t('agent.success'))
    navigate(-1)
  } catch (err: any) {
    toast.error(err.message)
  } finally {
    setSubmitting(false)
  }
}
```

---

## ✅ Definition of Done
- [ ] Agent farmer selector loads from Firestore `farmers` (falls back to mock in dev)
- [ ] `submitAgentReport` saves `agent_reports` doc with all form fields
- [ ] `lastReportAt` updated on the farmer doc
- [ ] `npm run build` exits 0 (web + functions)

```bash
firebase deploy --only functions:submitAgentReport
git commit -m "feat(s4-03): wire agent report to Firestore — real farmer list + submitAgentReport function"
git push origin feature/s4-03-agent-report
```
