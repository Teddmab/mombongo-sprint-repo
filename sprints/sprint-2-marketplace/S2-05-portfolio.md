# S2-05 — Portfolio Screen

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-05 |
| Sprint | Sprint 2 — Marketplace |
| Branch | `feature/s2-05-portfolio` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S2-04 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | useInvestments hook, /portfolio screen, update HomeScreen KPIs with real data |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — useInvestments hook

Create `src/hooks/useInvestments.ts`:
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions, isDevMode } from '@/lib/firebase'
import { useAuth } from '@/hooks/useAuth'
import { investments as MOCK } from '@/data/mock'

export interface Investment {
  id: string
  investorId: string
  productId: string
  productName: string
  productIcon: string
  amountUsd: number
  roi: number
  status: 'active' | 'completed' | 'cancelled'
  paymentStatus: 'pending' | 'completed'
  harvestDate: { seconds: number }  // Firestore Timestamp
  investedAt: { seconds: number }
}

export function useInvestments() {
  const { user } = useAuth()
  return useQuery({
    queryKey: ['investments', user?.uid],
    queryFn: async () => {
      if (!user?.uid) return []
      if (isDevMode()) return MOCK as unknown as Investment[]
      const result = await httpsCallable<Record<string, never>, { investments: Investment[] }>(functions, 'getInvestments')({})
      return result.data.investments
    },
    enabled: !!user?.uid,
    staleTime: 60_000,
  })
}
```

> **`mombongo-functions` dependency**: `getInvestments` onCall — reads `investments` where `investorId == context.auth.uid`, ordered by `investedAt` desc, returns `{ investments: [...] }`.

export function usePortfolioStats(investments: Investment[]) {
  const active = investments.filter(i => i.status === 'active')
  const totalInvested = active.reduce((s, i) => s + i.amountUsd, 0)
  const estimatedReturn = active.reduce((s, i) => s + i.amountUsd * i.roi / 100, 0)
  const nextHarvest = active
    .map(i => i.harvestDate?.seconds * 1000)
    .filter(Boolean)
    .sort()[0]
  return { totalInvested, estimatedReturn, activeCount: active.length, nextHarvest }
}
```

### Step 2 — Portfolio screen

The `/portfolio` route (add to router if not already present). The screen layout:

**Header KPI strip** (3 cards):
- Total investi: `formatUsd(stats.totalInvested)`
- Rendement estimé: `formatUsd(stats.estimatedReturn)`
- Investissements actifs: `stats.activeCount`

**Investment cards list:**
```tsx
{investments.map(inv => (
  <Link key={inv.id} to={`/market/${inv.productId}`} className="...">
    <div className="w-11 h-11 rounded-xl bg-green-50 flex items-center justify-center text-xl">
      {inv.productIcon}
    </div>
    <div className="flex-1">
      <p className="font-bold text-[13px]">{inv.productName}</p>
      <p className="text-[11px] text-gray-500">
        {t('portfolio.harvest')}: {formatDate(inv.harvestDate)}
      </p>
      <ProgressBar pct={daysProgress(inv)} />
    </div>
    <div className="text-right">
      <p className="font-extrabold text-[13px]">{formatUsd(inv.amountUsd)}</p>
      <p className="text-[11px] text-green-600 font-bold">+{inv.roi}% ROI</p>
    </div>
  </Link>
))}
```

Helper:
```typescript
function daysProgress(inv: Investment): number {
  const start = inv.investedAt?.seconds * 1000
  const end = inv.harvestDate?.seconds * 1000
  const now = Date.now()
  return Math.min(100, Math.max(0, Math.round((now - start) / (end - start) * 100)))
}

function formatDate(ts: { seconds: number }): string {
  return new Date(ts.seconds * 1000).toLocaleDateString('fr-FR', { day: 'numeric', month: 'short' })
}
```

### Step 3 — Update HomeScreen to use real stats

In `DesktopHome` and `MobileHome`, replace the `userProfile?.totalInvestedUsd ?? 4850` fallback:

```typescript
const { data: investments = [] } = useInvestments()
const stats = usePortfolioStats(investments)

const totalInvested = formatUsd(stats.totalInvested || userProfile?.totalInvestedUsd || 0)
const totalEarned = formatUsd(stats.estimatedReturn || userProfile?.totalEarnedUsd || 0)
```

This way:
- If user has real investments → shows real data
- If user has no investments yet → shows 0 (truthful) instead of the $4,850 mock fallback

### Step 4 — i18n keys

```
portfolio.title       → "Mon portefeuille" / "My portfolio"
portfolio.harvest     → "Récolte" / "Harvest"
portfolio.empty       → "Aucun investissement actif" / "No active investments"
portfolio.emptyDesc   → "Explorez le marché pour investir." / "Explore the market to invest."
```

---

## Unit Tests — `src/hooks/__tests__/useInvestments.test.ts`

```typescript
vi.mock('@/lib/firebase', () => ({ functions: {}, isDevMode: vi.fn(() => false) }))
vi.mock('@/hooks/useAuth', () => ({ useAuth: () => ({ user: { uid: 'u1' } }) }))

describe('usePortfolioStats', () => {
  it('sums totalInvested from active investments', () => {
    const inv = [
      { amountUsd: 200, roi: 22, status: 'active', harvestDate: { seconds: 0 } },
      { amountUsd: 500, roi: 18, status: 'active', harvestDate: { seconds: 0 } },
    ] as Investment[]
    const stats = usePortfolioStats(inv)
    expect(stats.totalInvested).toBe(700)
    expect(stats.activeCount).toBe(2)
  })

  it('excludes cancelled investments', () => {
    const inv = [
      { amountUsd: 200, roi: 22, status: 'cancelled', harvestDate: { seconds: 0 } },
    ] as Investment[]
    expect(usePortfolioStats(inv).totalInvested).toBe(0)
  })
})
```

---

## ✅ Definition of Done
- [ ] `/portfolio` shows real investments from Firestore
- [ ] KPI cards show real totals (not hardcoded $4,850)
- [ ] HomeScreen KPIs (`totalInvested`, `totalEarned`) use real investment data
- [ ] Empty state shown when user has no investments
- [ ] `data-testid="portfolio-screen"` on root element
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s2-05): portfolio screen + real investment stats on home"
git push origin feature/s2-05-portfolio
```
