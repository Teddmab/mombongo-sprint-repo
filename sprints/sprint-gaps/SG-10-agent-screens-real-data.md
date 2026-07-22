# SG-10 — Agent Screens: Real Data Wiring (Home, Market, Bourse, Financement)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-10 |
| Sprint | Sprint Gaps 10 |
| Branch | `feature/sg-10-agent-screens-real-data` |
| Merges into | `dev` |
| Estimate | 8h |
| Dependencies | SG-05 (getAgentFarmers CF) |

---

## Context

SG-05 wired the report submission form and defined `getAgentFarmers`. This story wires the
remaining 4 agent screens that all share the same `agentFarmers` list from mock:

- `AgentHome.tsx` — KPI cards, farmer list, recent reports all from mock
- `AgentMarket.tsx` — farmer list from mock; `PublierPourAgriculteurModal` = setTimeout
- `AgentBourse.tsx` — farmer list from mock; all "Alerter" buttons = toast stub
- `AgentFinancement.tsx` — farmer list + reports from mock

**Key insight**: `getAgentFarmers` (defined in SG-05) is the single shared CF call. Once
implemented, all 4 screens benefit. The additional work here is per-screen wiring + new CFs
for market publish-on-behalf and alert sending.

---

## Shared Hook (upgrade from SG-05)

`src/hooks/useAgentFarmers.ts` — wire with `useQuery` to `getAgentFarmers` CF. Already specified
in SG-05; this story assumes it exists and uses it.

---

## Cloud Functions

### `getAgentKpis`
```typescript
// Returns KPI aggregations for the agent's dashboard
// { totalFarmers, farmersOnTrack, totalAreaHa, disbursedThisMonth, pendingReports }
export const getAgentKpis = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const [farmersSnap, reportsSnap] = await Promise.all([
    db.collection('users').where('agentId', '==', uid).where('role', '==', 'farmer').get(),
    db.collection('agent_reports').where('agentId', '==', uid)
      .where('status', '==', 'pending').get(),
  ])

  const farmers = farmersSnap.docs.map(d => d.data())
  const totalAreaHa = farmers.reduce((sum: number, f: any) => sum + (f.surfaceHa ?? 0), 0)
  const farmersOnTrack = farmers.filter((f: any) => f.reportStatus === 'on_track').length

  return {
    totalFarmers: farmersSnap.size,
    farmersOnTrack,
    totalAreaHa,
    pendingReports: reportsSnap.size,
  }
})
```

### `publishListingForFarmer`
Agents can publish a product listing on behalf of a farmer they manage:
```typescript
// Params: { farmerId, commodity, quantityKg, pricePerKgCdf, province, territory,
//           quality, availableFrom, availableUntil, description? }
export const publishListingForFarmer = functions.region('europe-west1').https.onCall(async (data, context) => {
  const agentUid = context.auth?.uid
  if (!agentUid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  // Verify the agent is assigned to this farmer
  const farmerSnap = await db.collection('users').doc(data.farmerId).get()
  if (!farmerSnap.exists) throw new functions.https.HttpsError('not-found', 'Agriculteur introuvable')
  const farmer = farmerSnap.data()!
  if (farmer.agentId !== agentUid)
    throw new functions.https.HttpsError('permission-denied', "Cet agriculteur n'est pas dans votre portefeuille")

  // Create listing identical to createProductListing but with farmerId as sellerId
  const ref = db.collection('product_listings').doc()
  await ref.set({
    ...data,
    sellerId: data.farmerId,
    sellerName: farmer.displayName ?? farmer.fullName ?? 'Agriculteur',
    publishedByAgentId: agentUid,
    status: 'active',
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  })

  return { listingId: ref.id }
})
```

### `sendFarmerAlert`
Agents tap "Alerter" on a farmer row to send a push notification to that farmer:
```typescript
// Params: { farmerId, message }
export const sendFarmerAlert = functions.region('europe-west1').https.onCall(async (data, context) => {
  const agentUid = context.auth?.uid
  if (!agentUid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { farmerId, message } = data
  const farmerSnap = await db.collection('users').doc(farmerId).get()
  if (!farmerSnap.exists) throw new functions.https.HttpsError('not-found', 'Agriculteur introuvable')
  const farmer = farmerSnap.data()!
  if (farmer.agentId !== agentUid)
    throw new functions.https.HttpsError('permission-denied', "Pas dans votre portefeuille")

  // Write to notifications collection (SG-08)
  await db.collection('notifications').add({
    userId: farmerId,
    type: 'system',
    title: "Message de votre agent terrain",
    body: message,
    read: false,
    data: { agentId: agentUid },
    createdAt: FieldValue.serverTimestamp(),
  })

  // Also push via FCM if tokens available
  const tokens: string[] = farmer.fcmTokens ?? []
  if (tokens.length > 0) {
    const { getMessaging } = await import('firebase-admin/messaging')
    await getMessaging().sendMulticast({
      tokens,
      notification: { title: "Message de votre agent terrain", body: message },
      data: { type: 'system', agentId: agentUid },
    })
  }

  return { ok: true }
})
```

---

## Web: AgentHome changes

Replace mock `agentFarmers`, `agentReports`, and KPI values:
```tsx
const { data: farmers = [], isLoading } = useAgentFarmers()
const { data: kpis } = useQuery({ queryKey: ['agent-kpis'], queryFn: ... })
```

KPI cards:
- "Agriculteurs suivis" → `kpis.totalFarmers`
- "En bonne voie" → `kpis.farmersOnTrack`
- "Surface totale" → `kpis.totalAreaHa` ha
- "Rapports en attente" → `kpis.pendingReports`

Recent farmers table → real `farmers` list (first 5, sorted by `lastVisit` desc).

---

## Web: AgentMarket changes

**`PublierPourAgriculteurModal`**: Replace setTimeout stub with `publishListingForFarmer` CF call.

The modal already has a farmer selector; wire it to real `useAgentFarmers()` for the dropdown.
On submit, call:
```typescript
httpsCallable(functions, 'publishListingForFarmer')({ farmerId, commodity, quantityKg, ... })
```

---

## Web: AgentBourse changes

Replace mock `bourseTicker` with `useProductListings()` (already built in S8-01).

**"Alerter" button** on each farmer row:
```tsx
const sendAlert = useMutation({
  mutationFn: (farmerId: string) =>
    httpsCallable(functions, 'sendFarmerAlert')({ farmerId, message: 'Vérifiez votre calendrier de récolte' })
})
// <button onClick={() => sendAlert.mutate(farmer.id)}>Alerter</button>
```
Show a toast confirmation: "Alerte envoyée à {farmer.name}".

---

## Web: AgentFinancement changes

Replace mock `agentFarmers` with `useAgentFarmers()`.
Replace mock `agentReports` with the `useQuery(['agent-reports'])` hook from SG-05.
Financing amounts per farmer come from `financing_applications` (add `getAgentFinancingOverview` CF if needed or compute client-side from the farmer list).

---

## Acceptance Criteria
- [ ] `getAgentKpis` CF returns real aggregated KPIs
- [ ] `publishListingForFarmer` validates agent-farmer relationship + creates listing
- [ ] `sendFarmerAlert` writes to `notifications` + FCM
- [ ] `AgentHome` KPI cards show real values (not hardcoded)
- [ ] `AgentHome` farmer list uses `useAgentFarmers()` (real data)
- [ ] `PublierPourAgriculteurModal` calls `publishListingForFarmer` CF (not setTimeout)
- [ ] `AgentBourse` "Alerter" button calls `sendFarmerAlert` CF + shows toast
- [ ] `AgentFinancement` farmer list uses `useAgentFarmers()` (real data)
- [ ] Loading skeleton on all agent screens while data fetches
- [ ] Dev mode: mock data used, no CF calls
