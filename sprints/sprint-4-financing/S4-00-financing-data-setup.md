# S4-00 — Financing — Data Setup: Farmers, Applications & Admin CRUD

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S4-00 |
| Sprint | Sprint 4 — Financing |
| Branch | `feature/s4-00-financing-data-setup` |
| Merges into | `dev` |
| Estimate | 2.5h |
| Dependencies | S2-00 (functions project initialized) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | 🔨 Active | Seed farmers + financing_applications + cultural_events; Firestore rules |
| `mombongo-admin` | 🔨 Active | AdminFarmers + AdminFinancing screens |
| `mombongo-web` | ✅ Done | — |

---

## mombongo-functions

### Step 1 — Seed farmers collection

Create `src/scripts/seedFinancing.ts`:

```typescript
import { db } from '../lib/admin'
import { FieldValue } from 'firebase-admin/firestore'

const farmers = [
  {
    name: 'Jean-Baptiste Kalonji',
    region: 'Kasaï Central',
    cropType: 'Maïs',
    farmSizeHa: 5.2,
    requestedAmountUsd: 800,
    disbursedAmountUsd: 0,
    status: 'pending',           // pending | approved | active | completed
    agentId: null,
    nextHarvestDate: new Date('2026-09-15'),
    photoUrl: '',
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    name: 'Marie Ngalula',
    region: 'Bandundu',
    cropType: 'Manioc',
    farmSizeHa: 3.0,
    requestedAmountUsd: 500,
    disbursedAmountUsd: 250,
    status: 'active',
    agentId: null,
    nextHarvestDate: new Date('2026-08-01'),
    photoUrl: '',
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    name: 'Pierre Mukendi',
    region: 'Katanga',
    cropType: 'Soja',
    farmSizeHa: 8.0,
    requestedAmountUsd: 1200,
    disbursedAmountUsd: 0,
    status: 'pending',
    agentId: null,
    nextHarvestDate: new Date('2026-10-20'),
    photoUrl: '',
    createdAt: FieldValue.serverTimestamp(),
  },
]

const financingApplications = [
  {
    farmerId: 'PLACEHOLDER',    // replaced after seeding farmers
    investorId: null,
    requestedAmountUsd: 500,
    disbursedAmountUsd: 250,
    tranches: [
      { amountUsd: 250, status: 'disbursed', disbursedAt: new Date('2026-04-01') },
      { amountUsd: 250, status: 'pending',   disbursedAt: null },
    ],
    status: 'active',
    cropType: 'Manioc',
    createdAt: FieldValue.serverTimestamp(),
  },
]

const culturalEvents = [
  { cropType: 'Maïs',   eventType: 'planting',  monthStart: 10, monthEnd: 11, description: 'Semis de maïs — début saison sèche' },
  { cropType: 'Maïs',   eventType: 'harvest',   monthStart: 3,  monthEnd: 4,  description: 'Récolte maïs' },
  { cropType: 'Manioc', eventType: 'planting',  monthStart: 9,  monthEnd: 10, description: 'Bouturage manioc' },
  { cropType: 'Manioc', eventType: 'harvest',   monthStart: 9,  monthEnd: 12, description: 'Récolte manioc — 12 mois après semis' },
  { cropType: 'Soja',   eventType: 'planting',  monthStart: 4,  monthEnd: 5,  description: 'Semis soja grande saison' },
  { cropType: 'Soja',   eventType: 'harvest',   monthStart: 8,  monthEnd: 9,  description: 'Récolte soja' },
  { cropType: 'Cacao',  eventType: 'harvest',   monthStart: 10, monthEnd: 2,  description: 'Grande récolte cacao (Oct – Fév)' },
]

async function seed() {
  const batch = db.batch()
  for (const f of farmers) batch.set(db.collection('farmers').doc(), f)
  for (const e of culturalEvents) batch.set(db.collection('cultural_events').doc(), e)
  await batch.commit()
  console.log('Financing seeded')
}
seed().catch(console.error)
```

### Step 2 — Firestore security rules additions

Add to `firestore.rules`:
```
match /farmers/{farmerId} {
  allow read: if request.auth != null;
  allow write: if isAdmin();
}

match /financing_applications/{appId} {
  allow read: if request.auth != null
    && (resource.data.investorId == request.auth.uid || isAdmin());
  allow create: if request.auth != null
    && request.resource.data.investorId == request.auth.uid;
  allow update: if isAdmin();
}

match /agent_reports/{reportId} {
  allow read: if isAdmin();
  allow create: if request.auth != null
    && request.resource.data.agentId == request.auth.uid;
}

match /cultural_events/{eventId} {
  allow read: if true;
  allow write: if isAdmin();
}
```

---

## mombongo-admin

### AdminFarmers screen

`src/pages/AdminFarmers.tsx` — table columns: name / region / cropType / farmSizeHa / requestedAmountUsd / status chip / actions.

Actions:
- **View** → drawer with farmer details + financing applications
- **Approve** → sets `status: 'approved'` in Firestore
- **Assign agent** → sets `agentId` to selected admin user

```typescript
const { data: snapshot } = useQuery({
  queryKey: ['admin-farmers'],
  queryFn: () => getDocs(query(collection(db, 'farmers'), orderBy('createdAt', 'desc'), limit(50))),
})
const farmers = snapshot?.docs.map(d => ({ id: d.id, ...d.data() })) ?? []
```

Status chips: `pending` (amber) / `approved` (blue) / `active` (green) / `completed` (gray).

### AdminFinancing screen

`src/pages/AdminFinancing.tsx` — lists all `financing_applications` with farmer name (joined client-side from farmers query).

**Disburse tranche button**: calls `updateDoc` to mark next pending tranche as `disbursed` and increments farmer `disbursedAmountUsd`.

```typescript
async function disburseTranche(appId: string, trancheIndex: number, amount: number) {
  const appRef = doc(db, 'financing_applications', appId)
  const snap = await getDoc(appRef)
  const tranches = snap.data()?.tranches ?? []
  tranches[trancheIndex].status = 'disbursed'
  tranches[trancheIndex].disbursedAt = serverTimestamp()
  await updateDoc(appRef, { tranches, disbursedAmountUsd: increment(amount) })
  await updateDoc(doc(db, 'farmers', snap.data()?.farmerId), {
    disbursedAmountUsd: increment(amount),
  })
}
```

Route: add `/financing` and `/farmers` to admin router.

---

## ✅ Definition of Done
- [ ] Seed script populates `farmers`, `cultural_events` in Firestore
- [ ] Admin `/farmers` lists farmers with approve + assign agent actions
- [ ] Admin `/financing` lists applications, disburse tranche works
- [ ] Firestore rules deployed
- [ ] `npm run build` exits 0 (admin)

```bash
git commit -m "feat(s4-00): financing data setup — seed script + admin CRUD"
git push origin feature/s4-00-financing-data-setup
```
