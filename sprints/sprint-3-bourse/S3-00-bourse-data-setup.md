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
| `mombongo-functions` | 🔨 Active | Seed bourse_opportunities + bourse_prices collections |
| `mombongo-admin` | 🔨 Active | AdminBourse screen — manage opportunities + publish price updates |
| `mombongo-web` | ✅ Done | — |

---

## mombongo-functions

### Step 1 — Seed bourse_opportunities

Create `src/scripts/seedBourse.ts`:

```typescript
import { db } from '../lib/admin'
import { FieldValue } from 'firebase-admin/firestore'

const opportunities = [
  {
    route: 'Kisangani → Kinshasa',
    commodity: 'Cacao',
    description: 'Transport fluvial de cacao certifié bio, 48h de transit.',
    targetCdf: 9_800_000,
    minInvestCdf: 10_000,
    roi: 18,
    durationDays: 5,
    status: 'open',
    departureDate: new Date('2026-06-15'),
    capacityKg: 2000,
    filledKg: 0,
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    route: 'Mbandaka → Matadi',
    commodity: 'Maïs',
    description: 'Convoi routier via RN1, chargement garanti.',
    targetCdf: 5_400_000,
    minInvestCdf: 10_000,
    roi: 14,
    durationDays: 3,
    status: 'open',
    departureDate: new Date('2026-06-18'),
    capacityKg: 5000,
    filledKg: 0,
    createdAt: FieldValue.serverTimestamp(),
  },
  // ... remaining from mock bourseOpportunities
]

const prices = [
  { productName: 'Cacao', priceCdfPerKg: 4900, change: +2.3, recordedAt: FieldValue.serverTimestamp() },
  { productName: 'Maïs',  priceCdfPerKg: 1080, change: -0.8, recordedAt: FieldValue.serverTimestamp() },
  { productName: 'Manioc', priceCdfPerKg: 620, change: +1.1, recordedAt: FieldValue.serverTimestamp() },
]

async function seed() {
  const batch = db.batch()
  for (const o of opportunities) batch.set(db.collection('bourse_opportunities').doc(), o)
  for (const p of prices)        batch.set(db.collection('bourse_prices').doc(), p)
  await batch.commit()
  console.log('Bourse seeded')
}
seed().catch(console.error)
```

---

## mombongo-admin

### AdminBourse screen

`src/pages/AdminBourse.tsx` — two tabs:

**Tab 1 — Opportunités**: table with columns route/commodity/status/targetCdf/departure. Actions: create, edit, toggle status (open → review → completed), delete.

**Tab 2 — Prix du marché**: table with productName/priceCdfPerKg/change/recordedAt. "Add price" button creates a new `bourse_prices` document (admin only).

```typescript
// Queries
useQuery({ queryKey: ['admin-bourse-opps'], queryFn: () => getDocs(query(collection(db, 'bourse_opportunities'), orderBy('createdAt', 'desc'))) })
useQuery({ queryKey: ['admin-bourse-prices'], queryFn: () => getDocs(query(collection(db, 'bourse_prices'), orderBy('recordedAt', 'desc'), limit(50))) })

// Mutations — same pattern as AdminProducts in S2-00
```

Price updates use a simple form: select commodity name from a dropdown, enter new price, submit → `addDoc` to `bourse_prices`.

---

## ✅ Definition of Done
- [ ] Seed script populates `bourse_opportunities` and `bourse_prices` in Firestore
- [ ] Admin `/bourse` lists opportunities and prices
- [ ] Admin can create and toggle status of an opportunity
- [ ] Admin can add a new price entry
- [ ] `npm run build` exits 0 (admin)

```bash
git commit -m "feat(s3-00): bourse data setup — seed script + admin CRUD"
```
