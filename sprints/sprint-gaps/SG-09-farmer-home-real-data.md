# SG-09 — Farmer Home Screen: Real Data Wiring

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-09 |
| Sprint | Sprint Gaps 09 |
| Branch | `feature/sg-09-farmer-home-real-data` |
| Merges into | `dev` |
| Estimate | 6h |
| Dependencies | S10-00 (exploitation/cultures data model) |

---

## Context

`AgricultorHome.tsx` shows a personalised dashboard for farmers — but all data is hardcoded:

- `cropTasks` (upcoming tasks list) — from `mock.ts`
- `farmerAlerts` (alert list) — from `mock.ts`
- `disbursed` / `target` (financing progress bar) — hardcoded numbers
- Assigned agent name/phone/photo — hardcoded string
- KPI cards (loans, revenue, surface) — hardcoded

The farmer checks this screen daily. Nothing here is real. This story wires all four areas.

---

## Cloud Functions

### `getFarmerHomeData`
Single aggregation CF to avoid 4 separate round-trips on home screen load:

```typescript
// Returns: { cultures, agent, financing, alerts }
export const getFarmerHomeData = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  // 1. Upcoming tasks from cultures (S10-00 data model)
  const culturesSnap = await db.collection('exploitations').doc(uid)
    .collection('cultures').where('status', '==', 'active').get()
  const cultures = culturesSnap.docs.map(d => ({ id: d.id, ...d.data() }))

  // 2. Assigned agent
  const farmerSnap = await db.collection('users').doc(uid).get()
  const farmerData = farmerSnap.data() ?? {}
  let agent = null
  if (farmerData.agentId) {
    const agentSnap = await db.collection('users').doc(farmerData.agentId).get()
    if (agentSnap.exists) {
      const a = agentSnap.data()!
      agent = { name: a.displayName, phone: a.phone ?? '', photo: a.photoURL ?? null }
    }
  }

  // 3. Active financing application
  const finSnap = await db.collection('financing_applications')
    .where('farmerId', '==', uid)
    .where('status', 'in', ['approved', 'disbursed'])
    .orderBy('createdAt', 'desc')
    .limit(1)
    .get()
  const financing = finSnap.empty ? null : { id: finSnap.docs[0].id, ...finSnap.docs[0].data() }

  // 4. Alerts (unread notifications of type relevant to farmers)
  const alertsSnap = await db.collection('notifications')
    .where('userId', '==', uid)
    .where('read', '==', false)
    .orderBy('createdAt', 'desc')
    .limit(5)
    .get()
  const alerts = alertsSnap.docs.map(d => ({ id: d.id, ...d.data() }))

  return { cultures, agent, financing, alerts }
})
```

---

## Web Hook

`src/hooks/useFarmerHome.ts`:
```typescript
export function useFarmerHome() {
  if (isDevMode()) {
    return {
      isLoading: false,
      cultures: MOCK_CULTURES,    // from mock.ts (add if not present)
      agent: MOCK_AGENT,
      financing: MOCK_FINANCING,
      alerts: MOCK_FARMER_ALERTS,
    }
  }
  return useQuery({
    queryKey: ['farmer-home'],
    queryFn: async () => {
      const fn = httpsCallable(functions, 'getFarmerHomeData')
      const res = await fn({})
      return res.data as FarmerHomeData
    },
    staleTime: 60_000,
  })
}
```

---

## Crop Tasks Logic (frontend)

Derive "upcoming tasks" from cultures — no CF needed:
```typescript
function getCropTasks(cultures: Culture[]): CropTask[] {
  const now = new Date()
  const month = now.getMonth() + 1  // 1-based
  return cultures.flatMap(c => {
    const tasks: CropTask[] = []
    const monthsToSemis = c.moisSemis >= month ? c.moisSemis - month : 12 - month + c.moisSemis
    const monthsToRecolte = c.moisRecolte >= month ? c.moisRecolte - month : 12 - month + c.moisRecolte
    if (monthsToSemis <= 2)
      tasks.push({ culture: c.commodity, icon: c.icon, action: 'Semis', dueMonth: c.moisSemis, urgency: monthsToSemis === 0 ? 'now' : 'soon' })
    if (monthsToRecolte <= 2)
      tasks.push({ culture: c.commodity, icon: c.icon, action: 'Récolte', dueMonth: c.moisRecolte, urgency: monthsToRecolte === 0 ? 'now' : 'soon' })
    return tasks
  })
}
```

---

## UI Changes in `AgricultorHome.tsx`

Replace all mock constants with `useFarmerHome()`:

```tsx
const { cultures, agent, financing, alerts, isLoading } = useFarmerHome()
const cropTasks = getCropTasks(cultures ?? [])
const disbursed = financing?.disbursedAmount ?? 0
const target = financing?.requestedAmount ?? 0
```

**Assigned agent card**:
```
Mon agent de terrain
───────────────────────────────
[📷]  Jean-Baptiste Kimba
      Agronome terrain · Kinshasa
      📞 +243 81 234 5678        [Appeler]
```
If `agent` is null → "Aucun agent assigné pour le moment"

**Financing progress**:
```
Financement actif
  50 000 FC / 200 000 FC décaissés   ████░░░░  25%
```
If `financing` is null → "Aucun financement actif" with a "Faire une demande →" CTA

**Crop tasks list** (replaces mock `cropTasks`):
```
À faire ce mois
🌽 Maïs — Semis   dans 2 mois
🍠 Manioc — Récolte  ce mois ← highlighted
```

**Alerts** (replaces mock `farmerAlerts`):
Render `alerts` from `useFarmerHome()`, same UI but real data.

**Empty states**: If isLoading → skeleton cards. If cultures.length === 0 → "Complétez votre profil d'exploitation →" CTA linking to `/exploitation`.

---

## Acceptance Criteria
- [ ] `getFarmerHomeData` CF returns cultures, agent, financing, alerts in one call
- [ ] Crop tasks derived from real culture data (moisSemis / moisRecolte)
- [ ] Agent card shows real assigned agent name + phone, or "aucun agent" empty state
- [ ] Financing progress bar shows real disbursed/requested, or "aucun financement" state
- [ ] Alerts list shows real unread notifications
- [ ] Loading skeleton shown while data fetches
- [ ] Dev mode: MOCK data for cultures, agent, financing, alerts
