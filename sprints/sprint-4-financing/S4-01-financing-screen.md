# S4-01 — Financing — Farmer List Screen

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S4-01 |
| Sprint | Sprint 4 — Financing |
| Branch | `feature/s4-01-financing-screen` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S4-00 (farmers seeded) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /financing — farmer cards from Firestore, filter by crop/region |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — useFarmers hook

Create `src/hooks/useFinancing.ts`:

```typescript
import { useQuery } from '@tanstack/react-query'
import { collection, query, where, orderBy, getDocs } from 'firebase/firestore'
import { db, isDevMode } from '@/lib/firebase'
import { farmers as MOCK_FARMERS } from '@/data/mock'

export interface Farmer {
  id: string
  name: string
  region: string
  cropType: string
  farmSizeHa: number
  requestedAmountUsd: number
  disbursedAmountUsd: number
  status: 'pending' | 'approved' | 'active' | 'completed'
  nextHarvestDate: { seconds: number }
  photoUrl: string
}

export function useFarmers(filters?: { cropType?: string; region?: string; status?: string }) {
  return useQuery({
    queryKey: ['farmers', filters],
    queryFn: async () => {
      if (isDevMode()) return MOCK_FARMERS as unknown as Farmer[]
      let q = query(
        collection(db, 'farmers'),
        where('status', 'in', ['approved', 'active']),
        orderBy('createdAt', 'desc')
      )
      const snap = await getDocs(q)
      let results = snap.docs.map(d => ({ id: d.id, ...d.data() } as Farmer))
      if (filters?.cropType) results = results.filter(f => f.cropType === filters.cropType)
      if (filters?.region)   results = results.filter(f => f.region === filters.region)
      return results
    },
    staleTime: 60_000,
  })
}
```

Add mock data to `src/data/mock.ts`:
```typescript
export const farmers: Partial<Farmer>[] = [
  {
    id: 'f1', name: 'Jean-Baptiste Kalonji', region: 'Kasaï Central',
    cropType: 'Maïs', farmSizeHa: 5.2, requestedAmountUsd: 800,
    disbursedAmountUsd: 0, status: 'approved',
    nextHarvestDate: { seconds: 1757980800 }, photoUrl: '',
  },
  {
    id: 'f2', name: 'Marie Ngalula', region: 'Bandundu',
    cropType: 'Manioc', farmSizeHa: 3.0, requestedAmountUsd: 500,
    disbursedAmountUsd: 250, status: 'active',
    nextHarvestDate: { seconds: 1754035200 }, photoUrl: '',
  },
  {
    id: 'f3', name: 'Pierre Mukendi', region: 'Katanga',
    cropType: 'Soja', farmSizeHa: 8.0, requestedAmountUsd: 1200,
    disbursedAmountUsd: 0, status: 'approved',
    nextHarvestDate: { seconds: 1761350400 }, photoUrl: '',
  },
]
```

### Step 2 — FinancingScreen layout

Route: `/financing` (already exists in router).

In `src/pages/FinancingScreen.tsx`, replace mock data:
```typescript
const [cropFilter, setCropFilter] = useState<string | undefined>()
const [regionFilter, setRegionFilter] = useState<string | undefined>()
const { data: farmers = [], isLoading } = useFarmers({ cropType: cropFilter, region: regionFilter })
```

**Filter row** (horizontal scroll chips): unique crop types derived from farmers list. Active chip highlighted in green.

**Farmer cards** — each shows:
- Avatar circle with farmer initials (or `photoUrl` if set)
- Name + region
- Crop type chip
- Progress bar: `disbursedAmountUsd / requestedAmountUsd`
- `formatUsd(disbursedAmountUsd)` / `formatUsd(requestedAmountUsd)`
- Status chip: `approved` (blue, "En attente de financement") / `active` (green, "Financé") / `completed` (gray)
- Harvest date: `t('financing.harvest'): {formatDate(farmer.nextHarvestDate)}`
- "Financer" button → navigates to `/financing/:id`

**Empty state** when `farmers.length === 0`:
```tsx
<p className="text-center text-gray-500 mt-16">{t('financing.empty')}</p>
```

### Step 3 — i18n keys

```
financing.title       → "Financement agricole" / "Agricultural financing"
financing.filter      → "Filtrer par culture" / "Filter by crop"
financing.harvest     → "Récolte prévue" / "Expected harvest"
financing.funded      → "Financé" / "Funded"
financing.fund        → "Financer" / "Fund"
financing.empty       → "Aucun agriculteur disponible" / "No farmers available"
financing.progress    → "Progrès du financement" / "Funding progress"
```

---

## Unit Tests — `src/pages/__tests__/FinancingScreen.test.tsx`

```typescript
vi.mock('@/hooks/useFinancing', () => ({
  useFarmers: vi.fn(),
}))

it('renders farmer cards', () => {
  vi.mocked(useFarmers).mockReturnValue({
    data: [{ id: 'f1', name: 'Jean-Baptiste Kalonji', cropType: 'Maïs',
              region: 'Kasaï', requestedAmountUsd: 800, disbursedAmountUsd: 0,
              status: 'approved' }],
    isLoading: false,
  } as any)
  render(<FinancingScreen />)
  expect(screen.getByText(/Jean-Baptiste/)).toBeInTheDocument()
})

it('shows empty state when no farmers', () => {
  vi.mocked(useFarmers).mockReturnValue({ data: [], isLoading: false } as any)
  render(<FinancingScreen />)
  expect(screen.getAllByText(/Aucun agriculteur/).length).toBeGreaterThan(0)
})
```

---

## ✅ Definition of Done
- [ ] Farmer list loads from Firestore `farmers`
- [ ] Filter chips narrow results by crop type client-side
- [ ] `data-testid="financing-screen"` on root element
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s4-01): financing screen — farmer list from Firestore"
git push origin feature/s4-01-financing-screen
```
