# S3-01 — Bourse — Real-Time Commodity List

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S3-01 |
| Sprint | Sprint 3 — Bourse |
| Branch | `feature/s3-01-bourse-screen` |
| Merges into | `dev` |
| Estimate | 2.5h |
| Dependencies | S3-00 (bourse data seeded) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /bourse — live price ticker + opportunity list from Firestore |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — useBourse hooks

Create `src/hooks/useBourse.ts`:

```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions, isDevMode } from '@/lib/firebase'
import { bourseOpportunities as MOCK_OPPS, bourseTicker as MOCK_PRICES } from '@/data/mock'

// No Firestore SDK — all reads through Cloud Functions (db not exported from firebase.ts)

export interface BourseOpportunity {
  id: string
  route: string
  commodity: string
  description: string
  targetCdf: number
  minInvestCdf: number
  roi: number
  durationDays: number
  status: 'open' | 'review' | 'completed'
  departureDate: { seconds: number }
  capacityKg: number
  filledKg: number
}

export interface BoursePrice {
  id: string
  productName: string
  priceCdfPerKg: number
  change: number
  recordedAt: { seconds: number }
}

export function useBourseOpportunities() {
  return useQuery({
    queryKey: ['bourse-opportunities'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_OPPS as unknown as BourseOpportunity[]
      const result = await httpsCallable<Record<string, never>, { opportunities: BourseOpportunity[] }>(functions, 'getBourseOpportunities')({})
      return result.data.opportunities
    },
    staleTime: 60_000,
  })
}

// Prices polled via React Query refetch (no onSnapshot — all access through Functions)
export function useBoursePrices() {
  return useQuery({
    queryKey: ['bourse-prices'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_PRICES as unknown as BoursePrice[]
      const result = await httpsCallable<Record<string, never>, { prices: BoursePrice[] }>(functions, 'getBoursePrices')({})
      // Keep only latest price per productName
      const latest = new Map<string, BoursePrice>()
      result.data.prices.forEach(p => { if (!latest.has(p.productName)) latest.set(p.productName, p) })
      return [...latest.values()]
    },
    staleTime: 30_000,
    refetchInterval: 30_000,  // poll every 30 s instead of onSnapshot
  })
}
```

> **`mombongo-functions` dependencies**: `getBourseOpportunities` onCall (open opportunities ordered by departure), `getBoursePrices` onCall (latest 20 price docs).

### Step 2 — Wire BourseScreen

In `src/pages/BourseScreen.tsx`, replace mock data:
```typescript
const { data: opportunities = [], isLoading } = useBourseOpportunities()
const prices = useBoursePrices()
```

**Ticker strip** (horizontal scroll, auto-scrolling):
- Each item: `{price.productName} {formatCdf(price.priceCdfPerKg)}/kg`
- Color: green if `change > 0`, red if `change < 0`
- Shows live from `onSnapshot`

**Opportunity cards** — each shows:
- Route (origin → destination), departure date, commodity
- ROI badge, duration, min invest in CDF
- Progress bar: `filledKg / capacityKg`
- Status chip: `open` (green) / `review` (amber) / `completed` (gray)
- "Investir" button → navigates to `/bourse/:id`

### Step 3 — i18n keys

```
bourse.ticker      → "Prix du marché en direct" / "Live market prices"
bourse.departure   → "Départ" / "Departure"
bourse.capacity    → "Capacité" / "Capacity"
bourse.filled      → "rempli" / "filled"
bourse.invest      → "Investir" / "Invest"
bourse.empty       → "Aucune opportunité ouverte" / "No open opportunities"
```

---

## Unit Tests — `src/pages/__tests__/BourseScreen.test.tsx`

```typescript
vi.mock('@/hooks/useBourse', () => ({
  useBourseOpportunities: vi.fn(),
  useBoursePrices: vi.fn(() => []),
}))

it('renders opportunity cards', () => {
  vi.mocked(useBourseOpportunities).mockReturnValue({
    data: [{ id: 'b1', route: 'Kisangani → Kinshasa', commodity: 'Cacao', roi: 18, minInvestCdf: 10000, status: 'open' }],
    isLoading: false,
  } as any)
  render_()
  expect(screen.getByText(/Kisangani/)).toBeInTheDocument()
})
```

---

## ✅ Definition of Done
- [ ] Opportunity list loads from Firestore `bourse_opportunities`
- [ ] Price ticker reflects latest `bourse_prices` docs in real time
- [ ] `data-testid="bourse-screen"` on root
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s3-01): bourse screen wired to Firestore — live prices + opportunities"
git push origin feature/s3-01-bourse-screen
```
