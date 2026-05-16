# S1-03 — Home Screen (Investor View)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S1-03 |
| Sprint | Sprint 1 — Auth & Home |
| Branch | `feature/s1-03-home-screen` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 2 hours |
| Dependencies | S1-02, S0-03 (AppShell, AppHeader) |

---

## Step 1 — Home Screen UI

### Lovable Prompt
```
Replace HomeScreen stub with a real implementation.

Uses AppShell + AppHeader (showNotification, showLanguage, no title, no back).

HERO SECTION (bg-[#1E6B3F], px-4, pt-6 pb-12):
  "Bonjour, {userProfile.fullName.split(' ')[0]} 👋" (text-white, 22px, font-semibold)
  "Votre portefeuille" (text-white/70, 13px, mt-1)
  Total invested (text-white, 36px, font-bold, mt-1): formatUsd(userProfile.totalInvestedUsd)
  Two stat pills (mt-3, flex gap-2):
    "+{formatUsd(totalEarnedUsd)} gains" — bg-white/20 text-white rounded-full px-3 py-1 text-sm
    "5 investissements actifs" — same style

QUICK ACTIONS (white card, -mt-6, mx-4, rounded-2xl, shadow-md, p-4):
  Row of 4 action cards (grid grid-cols-4 gap-2):
  Each card: flex-col items-center, bg-gray-50 rounded-xl py-3 px-2
    Icon (24px, lucide-react) + label (text-xs, mt-1, text-gray-600, text-center)
  Cards:
    ShoppingBag "Investir" → navigate('/market')
    Briefcase "Portefeuille" → navigate('/portfolio')
    BarChart3 "Bourse" → navigate('/bourse')
    GraduationCap "Academia" → navigate('/academia')
  All in text-[#1E6B3F] color.

FEATURED PRODUCTS (mt-6, px-4):
  Header: "Opportunités du moment" (font-semibold, 16px) + "Voir tout →" link to /market
  Horizontal scroll (flex overflow-x-auto gap-3 pb-2 no-scrollbar):
    3 compact ProductMiniCards (220px wide each, flex-shrink-0)
    ProductMiniCard: white card, rounded-xl, p-3, shadow-sm
      iconEmoji (24px) + name (font-medium 13px) + location (text-gray-400 11px)
      ROI badge (green bg-[#EBF5EE] text-[#1E6B3F] text-xs rounded-full px-2 py-0.5): "{roi}% ROI"
      Mini progress bar (h-1.5, mt-2, green fill)
  Data: use useProducts() from TanStack Query, take first 3 with isFeatured:true
  Loading: 3 skeleton cards (SkeletonLoader variant="card" className="w-[220px] h-32")

RECENT ACTIVITY (mt-6, px-4, pb-6):
  Header: "Activité récente" + "Voir tout →" link to /portfolio
  If no investments: empty state "Pas encore d'activité. Commencez à investir!"
  If investments: list of up to 3 recent transactions (data from user's investments)
    Each row: ArrowUpRight icon (green) + "Investissement dans {productName}"
              + amount right-aligned + date below in gray text

OFFLINE BANNER (top of screen, below AppHeader):
  Amber banner when !navigator.onLine:
  "⚡ Données en cache" (text-sm)
  Disappears when connection restored (listen to 'online'/'offline' events)
```

### Unit Tests
File: `src/pages/__tests__/HomeScreen.test.tsx`
```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import HomeScreen from '@/pages/HomeScreen'

vi.mock('@/hooks/useAuth', () => ({
  useAuth: () => ({
    userProfile: { fullName: 'Alain Mutombo', totalInvestedUsd: 4850, totalEarnedUsd: 820, role: 'investor' },
    isAuthenticated: true, isLoading: false,
  })
}))
vi.mock('@/hooks/useProducts', () => ({
  useProducts: () => ({ data: [], isLoading: false })
}))

const render_ = () => render(
  <QueryClientProvider client={new QueryClient()}>
    <MemoryRouter><HomeScreen /></MemoryRouter>
  </QueryClientProvider>
)

describe('HomeScreen', () => {
  it('shows user first name greeting', () => {
    render_()
    expect(screen.getByText(/Bonjour, Alain/)).toBeInTheDocument()
  })

  it('shows total invested amount', () => {
    render_()
    expect(screen.getByText(/4.850|4,850|\$4/)).toBeInTheDocument()
  })

  it('renders 4 quick action cards', () => {
    render_()
    expect(screen.getByText(/Investir/)).toBeInTheDocument()
    expect(screen.getByText(/Portefeuille/)).toBeInTheDocument()
    expect(screen.getByText(/Bourse/)).toBeInTheDocument()
    expect(screen.getByText(/Academia/)).toBeInTheDocument()
  })

  it('shows featured products section', () => {
    render_()
    expect(screen.getByText(/Opportunités du moment/)).toBeInTheDocument()
  })
})
```

### Regression
```bash
bun run test:unit -- src/pages/__tests__/HomeScreen.test.tsx
# Expected: 4 tests pass
bun run build
```

📝 Manual: login → home shows your name, $4,850, 4 quick actions.

---

## ✅ Sprint 1 Complete 🎉
- [ ] Language → Auth → Home full flow working
- [ ] All unit tests pass
- [ ] `bun run build` exits 0

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0
- [ ] Offline banner works (DevTools → Network → Offline)
- [ ] Featured products load from Firestore

```bash
git add -A
git commit -m "feat(s1-03): home screen with portfolio stats, quick actions, featured products"
git push origin feature/s1-03-home-screen
# PR → dev → merge
# PR dev → staging → GitHub Actions deploys + E2E tests
```
