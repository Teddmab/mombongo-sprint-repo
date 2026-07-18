# S6-04 — Admin — Enhanced KPI Dashboard

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S6-04 |
| Sprint | Sprint 6 — Payments & Notifications |
| Branch | `feature/s6-04-admin-kpi-dashboard` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S6-03 (all collections populated) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-admin` | 🔨 Active | AdminDashboard — real-time KPIs, recharts bar chart, live activity feed |
| `mombongo-functions` | ✅ Done | — |
| `mombongo-web` | ✅ Done | — |

---

## mombongo-admin

### Step 1 — Real-time KPI listener

Replace the `getDashboardKpis()` polling in `admin.service.ts` with an `onSnapshot` listener so KPIs update in real time.

Create `src/hooks/useAdminKpis.ts`:

```typescript
import { useEffect, useState } from 'react'
import { collection, query, where, onSnapshot, getCountFromServer, getDocs, orderBy, limit } from 'firebase/firestore'
import { db } from '@/lib/firebase'

export interface AdminKpis {
  activeUsers:      number
  pendingKyc:       number
  monthlyVolumeUsd: number
  financingOpen:    number
  bourseOpen:       number
  totalDepositsUsd: number
}

export function useAdminKpis(): AdminKpis {
  const [kpis, setKpis] = useState<AdminKpis>({
    activeUsers: 0, pendingKyc: 0, monthlyVolumeUsd: 0,
    financingOpen: 0, bourseOpen: 0, totalDepositsUsd: 0,
  })

  useEffect(() => {
    async function fetchSnapshot() {
      const [activeSnap, kycSnap, financingSnap, bourseSnap] = await Promise.all([
        getCountFromServer(query(collection(db, 'users'), where('disabled', '!=', true))),
        getCountFromServer(query(collection(db, 'users'), where('kycStatus', '==', 'pending'))),
        getCountFromServer(query(collection(db, 'financing_applications'), where('status', '==', 'active'))),
        getCountFromServer(query(collection(db, 'bourse_opportunities'), where('status', '==', 'open'))),
      ])

      const startOfMonth = new Date()
      startOfMonth.setDate(1); startOfMonth.setHours(0, 0, 0, 0)
      const txSnap = await getDocs(
        query(collection(db, 'transactions'), where('createdAt', '>=', startOfMonth))
      )
      const monthlyVolumeUsd = txSnap.docs.reduce((s, d) => s + (d.data().amountUsd ?? 0), 0)

      const depositSnap = await getDocs(
        query(collection(db, 'deposits'), where('status', '==', 'completed'))
      )
      const totalDepositsUsd = depositSnap.docs.reduce((s, d) => s + (d.data().amountUsd ?? 0), 0)

      setKpis({
        activeUsers:      activeSnap.data().count,
        pendingKyc:       kycSnap.data().count,
        financingOpen:    financingSnap.data().count,
        bourseOpen:       bourseSnap.data().count,
        monthlyVolumeUsd,
        totalDepositsUsd,
      })
    }

    // Initial fetch + polling every 60s (real-time alternative to onSnapshot for aggregates)
    fetchSnapshot()
    const interval = setInterval(fetchSnapshot, 60_000)
    return () => clearInterval(interval)
  }, [])

  return kpis
}
```

### Step 2 — Monthly volume bar chart

Add recharts `BarChart` to `AdminDashboard.tsx` showing the last 6 months of `monthlyVolumeUsd`:

```typescript
export function useMonthlyVolume() {
  return useQuery({
    queryKey: ['admin-monthly-volume'],
    queryFn: async () => {
      const months: { month: string; volumeUsd: number }[] = []
      for (let i = 5; i >= 0; i--) {
        const start = new Date(); start.setMonth(start.getMonth() - i); start.setDate(1); start.setHours(0,0,0,0)
        const end   = new Date(start); end.setMonth(end.getMonth() + 1)
        const snap = await getDocs(
          query(
            collection(db, 'transactions'),
            where('createdAt', '>=', start),
            where('createdAt', '<',  end),
            where('type', 'in', ['investment', 'bourse_investment', 'financing'])
          )
        )
        const volume = snap.docs.reduce((s, d) => s + (d.data().amountUsd ?? 0), 0)
        months.push({
          month: start.toLocaleDateString('fr-FR', { month: 'short' }),
          volumeUsd: volume,
        })
      }
      return months
    },
    staleTime: 300_000,
  })
}
```

```tsx
// In AdminDashboard.tsx
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts'

const { data: monthlyData = [] } = useMonthlyVolume()

<ResponsiveContainer width="100%" height={200}>
  <BarChart data={monthlyData} margin={{ top: 5, right: 10, left: 0, bottom: 5 }}>
    <XAxis dataKey="month" tick={{ fontSize: 12 }} />
    <YAxis tick={{ fontSize: 12 }} tickFormatter={v => `$${v}`} />
    <Tooltip formatter={(v: number) => [`$${v.toFixed(0)}`, 'Volume']} />
    <Bar dataKey="volumeUsd" fill="#16a34a" radius={[4, 4, 0, 0]} />
  </BarChart>
</ResponsiveContainer>
```

### Step 3 — Live activity feed

Add a real-time `onSnapshot` feed of the 20 most recent transactions:

```typescript
export function useActivityFeed() {
  const [feed, setFeed] = useState<any[]>([])
  useEffect(() => {
    const q = query(collection(db, 'transactions'), orderBy('createdAt', 'desc'), limit(20))
    return onSnapshot(q, snap => {
      setFeed(snap.docs.map(d => ({ id: d.id, ...d.data() })))
    })
  }, [])
  return feed
}
```

```tsx
// Activity feed list
<ul className="divide-y">
  {feed.map(tx => (
    <li key={tx.id} className="py-2 flex items-center gap-3">
      <span className="text-lg">
        {tx.type === 'investment' ? '🌿'
          : tx.type === 'bourse_investment' ? '🚂'
          : tx.type === 'financing' ? '🌾'
          : tx.type === 'deposit' ? '💳'
          : '💰'}
      </span>
      <div className="flex-1">
        <p className="text-sm font-semibold capitalize">{tx.type.replace('_', ' ')}</p>
        <p className="text-xs text-gray-400">{tx.userId?.slice(0, 8)}…</p>
      </div>
      <span className="font-bold text-sm">{formatUsd(tx.amountUsd ?? 0)}</span>
    </li>
  ))}
</ul>
```

### Step 4 — KPI card updates

Update the 6 KPI cards in `AdminDashboard` to use `useAdminKpis()`:

```tsx
const kpis = useAdminKpis()

const KPI_CARDS = [
  { label: 'Utilisateurs actifs', value: kpis.activeUsers,      icon: '👥', color: 'blue'   },
  { label: 'KYC en attente',       value: kpis.pendingKyc,       icon: '🪪', color: 'amber'  },
  { label: 'Volume mensuel',       value: formatUsd(kpis.monthlyVolumeUsd), icon: '💵', color: 'green' },
  { label: 'Financements actifs',  value: kpis.financingOpen,    icon: '🌾', color: 'green'  },
  { label: 'Bourse ouverte',       value: kpis.bourseOpen,       icon: '🚂', color: 'purple' },
  { label: 'Total dépôts',         value: formatUsd(kpis.totalDepositsUsd), icon: '💳', color: 'teal' },
]
```

---

## ✅ Definition of Done
- [ ] KPI cards show real data from Firestore (refreshed every 60s)
- [ ] Bar chart shows 6 months of investment volume
- [ ] Activity feed updates in real time via `onSnapshot`
- [ ] `data-testid="admin-dashboard"` on root (already set in S1-03 admin work)
- [ ] `npm run build` exits 0 (admin)

```bash
git commit -m "feat(s6-04): admin dashboard — real-time KPIs, bar chart, activity feed"
git push origin feature/s6-04-admin-kpi-dashboard
```
