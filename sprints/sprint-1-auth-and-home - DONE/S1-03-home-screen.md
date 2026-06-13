# S1-03 — Home Screen (Investor View) / Admin Dashboard

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S1-03 |
| Sprint | Sprint 1 — Auth & Home |
| Branch | `feature/s1-03-home-screen` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 3 hours |
| Dependencies | S1-02, S0-03 (AppShell) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | useProducts hook + real data + offline banner |
| `mombongo-admin` | 🔨 Active | Connect adminService to Firestore + data-testid on dashboard |
| `mombongo-mobile` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-functions` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-backoffice` | ⏳ Sprint 2 | Repo not yet initialized |

---

## mombongo-web

### Current State (already implemented — do NOT rewrite)

`src/pages/HomeScreen.tsx` already contains a full desktop + mobile UI:
- ✅ Desktop: KPI strip (4 cards), portfolio area chart (recharts), recommended products grid, active investments table, recent activity feed
- ✅ Mobile: portfolio card, quick-action grid (4 icons), active investments list, recent activity
- ✅ Farmer role variant: `FarmerHome` component (separate layout for `role === "farmer"`)
- ✅ `data-testid="home-screen"` on root
- ✅ `formatUsd`, `formatCdf`, `formatPercent` helpers available in `@/lib/utils`
- ✅ `useIsDesktop()` responsive dispatch

**What is NOT connected to real data yet (this sprint's work):**
- ❌ Portfolio values are hardcoded: `"$4,850.00"`, `"$342"`, `"3"`, etc.
- ❌ User name comes from `useApp().userName` (always `"Alain"`) not from `useAuth().userProfile`
- ❌ Product cards use static `products` array from `@/data/mock` — not a Query hook
- ❌ No offline banner
- ❌ No `useProducts()` hook

**Do NOT change the UI layout, colors, or component structure. Only swap data sources.**

---

## Step 1 — useProducts hook

Create `src/hooks/useProducts.ts` using TanStack Query.

```typescript
import { useQuery } from '@tanstack/react-query'
import { collection, getDocs, query, where, orderBy, limit } from 'firebase/firestore'
import { db, isDevMode } from '@/lib/firebase'
import { products as MOCK_PRODUCTS } from '@/data/mock'

export interface Product {
  id: string
  name: string
  location: string
  farmer: string
  icon: string
  category: string
  roi: number
  minInvest: number
  duration: string
  isFeatured: boolean
  progress: number
  // add other fields that exist in @/data/mock products array
}

async function fetchProducts(): Promise<Product[]> {
  if (isDevMode()) return MOCK_PRODUCTS as unknown as Product[]
  const snap = await getDocs(
    query(collection(db, 'products'), where('isActive', '==', true), orderBy('roi', 'desc'), limit(20))
  )
  return snap.docs.map(d => ({ id: d.id, ...d.data() } as Product))
}

export function useProducts() {
  return useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
    staleTime: 300_000,
  })
}

export function useFeaturedProducts() {
  const { data, ...rest } = useProducts()
  return { data: data?.filter(p => p.isFeatured).slice(0, 3) ?? [], ...rest }
}
```

> **Note:** `isDevMode()` returns `true` in the local dev environment (`VITE_DEV_MODE=true`), so the hook falls back to mock data — the UI keeps working without a Firestore connection.

---

## Step 2 — Connect real data in HomeScreen

**Do not restructure the components.** Make targeted replacements inside the existing `MobileHome` and `DesktopHome` functions.

### 2a — User name from useAuth

Replace `useApp().userName` with `useAuth().userProfile?.displayName` for the greeting. Add the import at the top of the file:

```tsx
import { useAuth } from '@/hooks/useAuth'
```

Inside `DesktopHome` and `MobileHome`, add:
```tsx
const { userProfile } = useAuth()
const firstName = userProfile?.displayName?.split(' ')[0] ?? userName
```

Then replace any hard-reference to `userName` in the greeting text with `firstName`.

### 2b — Portfolio values from userProfile

Replace the hardcoded `"$4,850.00"` and `"$342"` with:
```tsx
import { formatUsd } from '@/lib/utils'

// In the component:
const totalInvested = formatUsd(userProfile?.totalInvestedUsd ?? 4850)
const totalEarned   = formatUsd(userProfile?.totalEarnedUsd  ?? 342)
```

Substitute `totalInvested` and `totalEarned` where the hardcoded strings appear.

### 2c — Featured products from useFeaturedProducts

In `MobileHome` and `DesktopHome`, replace `products.slice(...)` with the hook:

```tsx
import { useFeaturedProducts } from '@/hooks/useProducts'
import { SkeletonLoader } from '@/components/ui/SkeletonLoader'

// Inside the component:
const { data: featured, isLoading: productsLoading } = useFeaturedProducts()
```

Where the product cards render, wrap with a loading state:
```tsx
{productsLoading
  ? Array.from({ length: 3 }).map((_, i) => <SkeletonLoader key={i} variant="card" className="w-[220px] h-32" />)
  : featured.map(p => <ProductMiniCard key={p.id} product={p} />)
}
```

### 2d — Offline banner

Add this component at the top of **both** `MobileHome` and `DesktopHome`, directly below any header section:

```tsx
function OfflineBanner() {
  const [offline, setOffline] = useState(!navigator.onLine)
  useEffect(() => {
    const on  = () => setOffline(false)
    const off = () => setOffline(true)
    window.addEventListener('online', on)
    window.addEventListener('offline', off)
    return () => { window.removeEventListener('online', on); window.removeEventListener('offline', off) }
  }, [])
  if (!offline) return null
  return (
    <div className="bg-amber-50 border-b border-amber-200 px-4 py-2 text-amber-800 text-[13px] flex items-center gap-2">
      <span>⚡</span> Données en cache · Reconnexion en cours…
    </div>
  )
}
```

---

## Unit Tests
File: `src/pages/__tests__/HomeScreen.test.tsx`

```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import HomeScreen from '@/pages/HomeScreen'

vi.mock('@/hooks/useAuth', () => ({
  useAuth: () => ({
    userProfile: {
      displayName: 'Alain Mutombo',
      totalInvestedUsd: 4850,
      totalEarnedUsd: 820,
      role: 'investor',
    },
    isAuthenticated: true,
    isLoading: false,
  })
}))
vi.mock('@/hooks/useProducts', () => ({
  useProducts: () => ({ data: [], isLoading: false }),
  useFeaturedProducts: () => ({ data: [], isLoading: false }),
}))
vi.mock('@/context/AppContext', () => ({
  useApp: () => ({ role: 'investor', userName: 'Alain', lang: 'fr', setLang: vi.fn(), setRole: vi.fn() }),
  AppProvider: ({ children }: any) => children,
}))

const render_ = () => render(
  <QueryClientProvider client={new QueryClient()}>
    <MemoryRouter><HomeScreen /></MemoryRouter>
  </QueryClientProvider>
)

describe('HomeScreen', () => {
  it('shows user first name from userProfile', () => {
    render_()
    expect(screen.getByText(/Alain/)).toBeInTheDocument()
  })

  it('shows total invested amount from userProfile', () => {
    render_()
    // formatUsd(4850) → "$4,850.00"
    expect(screen.getByText(/4.850|4,850|\$4/)).toBeInTheDocument()
  })

  it('renders quick action cards', () => {
    render_()
    expect(screen.getByText(/Investir|market|Marché/i)).toBeInTheDocument()
  })

  it('renders featured products section header', () => {
    render_()
    expect(screen.getByText(/Opportunités|moment|Featured/i)).toBeInTheDocument()
  })
})
```

### Regression
```bash
bun run test:unit -- src/pages/__tests__/HomeScreen.test.tsx
# Expected: 4 tests pass
bun run build
# Expected: exits 0
```

📝 Manual checklist:
- [ ] User name in greeting shows real first name after login
- [ ] Portfolio card shows `$4,850.00` (or real value from Firestore in staging)
- [ ] Product cards load from mock in dev, from Firestore in staging
- [ ] Offline: DevTools → Network → Offline → amber banner appears, restores when back online

---

## mombongo-admin

### Current State (already implemented — do NOT rewrite)

`src/pages/AdminDashboard.tsx` already has a full UI:
- ✅ 4 KPI stat cards (active users, pending KYC, monthly volume, financing open)
- ✅ Area chart (volume + approvals over a week)
- ✅ TanStack Query: `useQuery` calls `adminService.getDashboardKpis()` and `adminService.getActivity()`
- ✅ `formatUsd` from `@/lib/utils` for the volume card
- ✅ Framer Motion entrance animation

**What is NOT done yet:**
- ❌ `adminService` methods return hardcoded stub data — no Firestore queries
- ❌ No `data-testid` on the dashboard root element
- ❌ `src/services/admin.service.ts` has no real Firestore implementation

### Step 1 — Add data-testid to AdminDashboard

In `src/pages/AdminDashboard.tsx`, add `data-testid="admin-dashboard"` to the root `<motion.section>`:

```tsx
// Before
<motion.section className="page" ...>

// After
<motion.section data-testid="admin-dashboard" className="page" ...>
```

### Step 2 — Connect adminService to Firestore

Replace stub constants in `src/services/admin.service.ts` with real Firestore queries.
Import `db` from `@/lib/firebase` and query the collections defined in `firestore.rules`.

```typescript
import { collection, getDocs, query, where, getCountFromServer, orderBy, limit } from 'firebase/firestore'
import { db } from '@/lib/firebase'

export class AdminService {
  async getDashboardKpis(): Promise<DashboardKpis> {
    const [usersSnap, kycSnap, investSnap, financingSnap] = await Promise.all([
      getCountFromServer(query(collection(db, 'users'), where('isActive', '==', true))),
      getCountFromServer(query(collection(db, 'users'), where('kycStatus', '==', 'pending'))),
      getDocs(query(collection(db, 'investments'), where('paymentStatus', '==', 'completed'))),
      getCountFromServer(query(collection(db, 'farmer_financing'), where('status', '==', 'review'))),
    ])
    const monthlyVolumeUsd = investSnap.docs.reduce((sum, d) => sum + (d.data().amountUsd ?? 0), 0)
    return {
      activeUsers: usersSnap.data().count,
      pendingKyc: kycSnap.data().count,
      monthlyVolumeUsd,
      financingOpen: financingSnap.data().count,
    }
  }

  async getActivity(): Promise<ActivityPoint[]> {
    // Activity is derived data — return stub until a dedicated collection exists
    return [
      { name: 'Mon', volume: 18, approvals: 5 },
      { name: 'Tue', volume: 24, approvals: 8 },
      { name: 'Wed', volume: 21, approvals: 7 },
      { name: 'Thu', volume: 31, approvals: 10 },
      { name: 'Fri', volume: 29, approvals: 9 },
    ]
  }

  async getUsers(): Promise<AdminUserRow[]> {
    const snap = await getDocs(query(collection(db, 'users'), orderBy('createdAt', 'desc'), limit(50)))
    return snap.docs.map(d => ({
      id: d.id,
      name: d.data().fullName ?? '',
      email: d.data().email ?? '',
      role: d.data().role,
      status: d.data().isActive ? 'active' : 'suspended',
    } as AdminUserRow))
  }

  async getTransactions(): Promise<AdminTransactionRow[]> {
    const snap = await getDocs(query(collection(db, 'transactions'), orderBy('createdAt', 'desc'), limit(50)))
    return snap.docs.map(d => ({
      id: d.id,
      description: d.data().description ?? '',
      amountUsd: d.data().amountUsd ?? 0,
      status: d.data().status ?? 'pending',
      createdAt: d.data().createdAt?.toDate?.()?.toISOString() ?? '',
    } as AdminTransactionRow))
  }

  async getFinancingPipeline(): Promise<FinancingRow[]> {
    const snap = await getDocs(query(collection(db, 'farmer_financing'), orderBy('createdAt', 'desc'), limit(50)))
    return snap.docs.map(d => ({
      id: d.id,
      farmer: d.data().farmerName ?? '',
      crop: d.data().crop ?? '',
      requestedUsd: d.data().requestedUsd ?? 0,
      status: d.data().status ?? 'review',
    } as FinancingRow))
  }

  async getBoursePipeline(): Promise<BourseRow[]> {
    const snap = await getDocs(query(collection(db, 'bourse_opportunities'), orderBy('createdAt', 'desc'), limit(50)))
    return snap.docs.map(d => ({
      id: d.id,
      route: d.data().route ?? '',
      commodity: d.data().commodity ?? '',
      targetCdf: d.data().targetCdf ?? 0,
      status: d.data().status ?? 'open',
    } as BourseRow))
  }
}

export const adminService = new AdminService()
```

> **Note:** `getActivity()` stays as stub data — the `activity` pattern requires an aggregated time-series collection that is not yet defined in Firestore. This is tracked as a future sprint task.

### Regression
```bash
# Run from mombongo-admin/
bun run build
# Expected: exits 0
bun run test:unit
# Expected: existing LoginScreen tests pass
```

📝 Manual checklist:
- [ ] Admin dashboard KPI cards show real counts from Firestore (staging env)
- [ ] Users table loads real users
- [ ] Transactions table loads real transactions
- [ ] `data-testid="admin-dashboard"` visible in DevTools

---

## ✅ Sprint 1 Complete 🎉

### mombongo-web
- [ ] Language → Auth → Home full flow working end-to-end
- [ ] All unit tests pass (`bun run test:unit`)
- [ ] `bun run build` exits 0
- [ ] Offline banner works
- [ ] `useProducts` falls back to mock when `isDevMode() === true`
- [ ] No hardcoded `"$4,850"` or `"Alain"` strings remaining in HomeScreen

### mombongo-admin
- [ ] `data-testid="admin-dashboard"` on root section
- [ ] `adminService` queries Firestore (not stubs) for all methods except `getActivity()`
- [ ] All existing admin tests pass
- [ ] `bun run build` exits 0

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0 (web + admin)
- [ ] Offline banner tested in DevTools (web)
- [ ] Admin staging env has valid `.env` with Firebase config before testing Firestore queries

```bash
# mombongo-web
git add -A
git commit -m "feat(s1-03): connect home screen to real data — useProducts, userProfile, offline banner"
git push origin feature/s1-03-home-screen

# mombongo-admin
git add -A
git commit -m "feat(s1-03): connect adminService to Firestore + data-testid on dashboard"
git push origin feature/s1-03-admin-dashboard
# PR → dev → merge
```
