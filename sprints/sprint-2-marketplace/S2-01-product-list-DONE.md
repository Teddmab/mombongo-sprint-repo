# S2-01 — Marketplace — Product List & Search

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-01 |
| Sprint | Sprint 2 — Marketplace |
| Branch | `feature/s2-01-product-list` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S2-00 (products seeded in Firestore) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Wire /market investor view to Firestore via `useProducts`, add loading/empty/error states |
| `mombongo-admin` | ✅ Done | Products CRUD from S2-00 |
| `mombongo-functions` | ✅ Done | No new functions |

---

## mombongo-web

### Current State
`useProducts.ts` already exists (created in S1-03) and queries Firestore when `isDevMode()` is false, falling back to `src/data/mock.ts` in dev.

`MarketScreen.tsx` was extended in SP-01 with role-dispatched views — `AgricultorMarket`, `AgentMarket`, and `MerchantMarket` are separate components rendered by role. The **investor view** (`DesktopMarket` / `MobileMarket`) still imports `products` directly from mock instead of using the hook.

**What this story does:** wire the investor view's `DesktopMarket` and `MobileMarket` sub-components to `useProducts`, add skeleton / empty / error states, and cover with unit tests.

> Role-specific market views (AgricultorMarket, AgentMarket, MerchantMarket) each manage their own data independently and are not in scope here.

### Step 1 — Wire MarketScreen investor view to useProducts

In `src/pages/MarketScreen.tsx`, remove the static import and add the hook inside `DesktopMarket` and `MobileMarket`:

```typescript
// Remove:
import { products, Category } from '@/data/mock'

// Keep Category type (or define locally as string union):
import type { Category } from '@/data/mock'

// Add:
import { useProducts } from '@/hooks/useProducts'
import { SkeletonLoader } from '@/components/ui/SkeletonLoader'

// Inside DesktopMarket and MobileMarket:
const { data: products = [], isLoading, isError } = useProducts()
```

### Step 2 — Add loading / empty / error states

```tsx
// Loading — in the product grid section:
{isLoading && (
  <div className="grid grid-cols-3 xl:grid-cols-4 gap-4">
    {Array.from({ length: 8 }).map((_, i) => (
      <SkeletonLoader key={i} className="h-56 rounded-2xl" />
    ))}
  </div>
)}

// Error
{isError && (
  <p className="text-center py-12 text-red-500 text-[13px]">
    Erreur de chargement. Réessayez.
  </p>
)}

// Empty (not loading, no error, empty array)
{!isLoading && !isError && products.length === 0 && (
  <div className="flex flex-col items-center justify-center py-20 text-gray-400">
    <Sprout className="w-12 h-12 mb-3 opacity-40" />
    <p className="font-display font-bold text-[15px]">Aucun produit disponible</p>
    <p className="text-[13px] mt-1">Revenez bientôt.</p>
  </div>
)}
```

### Step 3 — Fix category counts in sidebar

The sidebar currently reads `products.filter(...)` at render time using the static array. After switching to the hook this still works since `products` is the reactive array from the hook (`const { data: products = [] } = useProducts()`). No change needed.

---

## Unit Tests — `src/pages/__tests__/MarketScreen.test.tsx`

```typescript
vi.mock('@/hooks/useProducts', () => ({ useProducts: vi.fn() }))

describe('MarketScreen — investor view', () => {
  it('renders skeletons while loading', () => {
    vi.mocked(useProducts).mockReturnValue({ data: [], isLoading: true, isError: false } as any)
    render(<MarketScreen />)
    // SkeletonLoader renders divs; check container presence
    expect(document.querySelector('[data-testid="market-screen"]')).toBeInTheDocument()
  })

  it('renders product cards when data loaded', () => {
    vi.mocked(useProducts).mockReturnValue({
      data: [{ id: 'p1', name: 'Pastèques', category: 'agriculture', roi: 22, minInvest: 200, duration: 30, location: 'Songololo', icon: '' }],
      isLoading: false, isError: false,
    } as any)
    render(<MarketScreen />)
    expect(screen.getByText('Pastèques')).toBeInTheDocument()
  })

  it('renders empty state when no products', () => {
    vi.mocked(useProducts).mockReturnValue({ data: [], isLoading: false, isError: false } as any)
    render(<MarketScreen />)
    expect(screen.getByText(/Aucun produit/i)).toBeInTheDocument()
  })
})
```

---

## ✅ Definition of Done
- [x] `useProducts.ts` exists (S1-03) ✅
- [ ] `DesktopMarket` and `MobileMarket` use `useProducts()` — not static mock import
- [ ] Skeleton shown while loading
- [ ] Empty state rendered when products array is empty
- [ ] `data-testid="market-screen"` present (already added in SP-01)
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s2-01): wire /market investor view to useProducts — loading, empty, error states"
git push origin feature/s2-01-product-list
```
