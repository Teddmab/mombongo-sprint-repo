# S3-00 — Bourse — Data Setup: Opportunities, Prices & Admin CRUD

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S3-00 |
| Sprint | Sprint 3 — Bourse |
| Branch | `feature/s3-00-bourse-data-setup` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S2-00 (functions project initialized) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | ⚠️ Partial | CFs done (S3-01/02/03); seed script still needed |
| `mombongo-admin` | 🔨 Active | AdminBourse screen — manage opportunities + publish price updates |
| `mombongo-web` | ✅ Done | — |

---

## Status update (2026-07-16)

The following Cloud Functions were added in S3-01/02/03 and are already in the codebase:
- `getBourseOpportunities` ✅
- `getBoursePrices` ✅
- `getBourseOpportunity` ✅
- `createBourseInvestment` ✅

**What still remains in this story:**
1. Seed script to populate `bourse_opportunities` and `bourse_prices` in Firestore
2. AdminBourse screen in `mombongo-admin`

---

## mombongo-functions

### Step 1 — Seed bourse_opportunities

Create `src/scripts/seedBourse.ts`:

```typescript
import { db } from '../lib/admin'
import { FieldValue } from 'firebase-admin/firestore'

const opportunities = [
  {
    title: 'Transport Tomates Matadi → Kinshasa',
    type: 'transport',
    origin: 'Matadi',
    destination: 'Kinshasa',
    volume: '120 bacs',
    price: '75,000 FC',
    commission: 20,
    duration: '5 jours',
    spotsLeft: 3,
    spotsTotal: 8,
    status: 'open',
    targetCdf: 9_800_000,
    minInvestCdf: 10_000,
    capacityKg: 2000,
    filledKg: 0,
    departureDate: new Date('2026-08-15'),
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    title: 'Stockage Manioc Kinshasa',
    type: 'stockage',
    origin: 'Kinshasa',
    volume: '200 sacs',
    price: '40,000 FC',
    commission: 12,
    duration: '30 jours',
    spotsLeft: 6,
    spotsTotal: 10,
    status: 'open',
    targetCdf: 5_400_000,
    minInvestCdf: 10_000,
    capacityKg: 5000,
    filledKg: 0,
    departureDate: new Date('2026-08-20'),
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    title: 'Transformation Café Kivu',
    type: 'transformation',
    origin: 'Goma',
    volume: '500 kg',
    price: '$1,200',
    commission: 28,
    duration: '21 jours',
    spotsLeft: 2,
    spotsTotal: 5,
    status: 'open',
    targetCdf: 3_000_000,
    minInvestCdf: 10_000,
    capacityKg: 500,
    filledKg: 0,
    departureDate: new Date('2026-08-18'),
    createdAt: FieldValue.serverTimestamp(),
  },
]

const prices = [
  { symbol: 'TOM-MAT', price: '1,250 FC/kg', change: 2.4, recordedAt: FieldValue.serverTimestamp() },
  { symbol: 'PAST-SGL', price: '850 FC/kg', change: -1.1, recordedAt: FieldValue.serverTimestamp() },
  { symbol: 'CAF-KIV', price: '$4.20/lb', change: 3.8, recordedAt: FieldValue.serverTimestamp() },
  { symbol: 'CAC-BC', price: '$3.10/kg', change: 1.6, recordedAt: FieldValue.serverTimestamp() },
  { symbol: 'MAN-KIN', price: '320 FC/kg', change: 0.5, recordedAt: FieldValue.serverTimestamp() },
  { symbol: 'OIG-KIN', price: '1,800 FC/kg', change: -0.7, recordedAt: FieldValue.serverTimestamp() },
]

async function seed() {
  const batch = db.batch()
  for (const o of opportunities) batch.set(db.collection('bourse_opportunities').doc(), o)
  for (const p of prices)        batch.set(db.collection('bourse_prices').doc(), p)
  await batch.commit()
  console.log('Bourse seeded ✓')
}
seed().catch(console.error)
```

Run: `npx ts-node src/scripts/seedBourse.ts`

---

## mombongo-admin

### AdminBourse screen

`src/pages/AdminBourse.tsx` — two tabs:

**Tab 1 — Opportunités**: table with columns title/type/status/spotsLeft/departure. Actions: create, edit, toggle status (open → review → completed), delete.

**Tab 2 — Prix du marché**: table with symbol/price/change/recordedAt. "Add price" button creates a new `bourse_prices` document (admin only).

Note: admin uses Firestore SDK directly (`db` from `src/lib/firebase.ts` in mombongo-admin) — this is allowed for admin, not for the user-facing `mombongo-web`.

```typescript
// Queries
useQuery({ queryKey: ['admin-bourse-opps'], queryFn: () => getDocs(query(collection(db, 'bourse_opportunities'), orderBy('createdAt', 'desc'))) })
useQuery({ queryKey: ['admin-bourse-prices'], queryFn: () => getDocs(query(collection(db, 'bourse_prices'), orderBy('recordedAt', 'desc'), limit(50))) })

// Mutations — same pattern as AdminProducts in S2-00
```

---

## ✅ Definition of Done
- [x] `getBourseOpportunities` CF deployed ← done in S3-01
- [x] `getBoursePrices` CF deployed ← done in S3-01
- [x] `getBourseOpportunity` CF deployed ← done in S3-02
- [x] `createBourseInvestment` CF deployed ← done in S3-03
- [ ] Seed script populates `bourse_opportunities` and `bourse_prices` in Firestore
- [ ] Admin `/bourse` lists opportunities and prices
- [ ] Admin can create and toggle status of an opportunity
- [ ] Admin can add a new price entry
- [ ] `npm run build` exits 0 (admin)

```bash
git commit -m "feat(s3-00): bourse data setup — seed script + admin CRUD"
```
