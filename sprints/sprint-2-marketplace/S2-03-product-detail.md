# S2-03 — Marketplace — Product Detail Screen

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-03 |
| Sprint | Sprint 2 — Marketplace |
| Branch | `feature/s2-03-product-detail` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S2-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /market/:id — fetch single product from Firestore, farmer info, invest CTA |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Current State
`/market/:id` screen exists (ProductDetailScreen) using static `products.find(p => p.id === id)`.

### Step 1 — useProduct hook

Add to `src/hooks/useProducts.ts`:
```typescript
import { doc, getDoc } from 'firebase/firestore'

export function useProduct(id: string) {
  return useQuery({
    queryKey: ['product', id],
    queryFn: async () => {
      if (isDevMode()) return MOCK_PRODUCTS.find(p => p.id === id) ?? null
      const snap = await getDoc(doc(db, 'products', id))
      return snap.exists() ? ({ id: snap.id, ...snap.data() } as Product) : null
    },
    enabled: !!id,
  })
}
```

### Step 2 — Wire ProductDetailScreen

```typescript
// In ProductDetailScreen:
const { id } = useParams<{ id: string }>()
const { data: product, isLoading } = useProduct(id!)
const navigate = useNavigate()

if (isLoading) return <SkeletonLoader variant="list" lines={6} className="max-w-2xl mx-auto mt-8" />
if (!product) return <NotFound />
```

### Step 3 — Screen layout

The product detail screen shows (desktop: two-column, mobile: single-column scroll):

**Left / Top section:**
- Product hero image (if `product.image` exists) or icon in a large coloured circle
- Product name, category badge, location with MapPin icon
- `isFeatured` badge if applicable

**Stats row:**
```tsx
<div className="grid grid-cols-3 gap-3">
  <StatCard label={t('product.roi')} value={`+${product.roi}%`} accent="green" />
  <StatCard label={t('product.duration')} value={`${product.duration}j`} accent="blue" />
  <StatCard label={t('product.minInvest')} value={`$${product.minInvest}`} accent="amber" />
</div>
```

**Funding progress bar:**
```tsx
const pct = Math.min(100, Math.round((product.invested ?? 0) / (product.stock * product.minInvest) * 100))
<div className="h-2 bg-gray-100 rounded-full">
  <div className="h-2 bg-green-700 rounded-full" style={{ width: `${pct}%` }} />
</div>
<p>{pct}% {t('product.funded')} · {product.stock - (product.investorsCount ?? 0)} {t('product.slotsLeft')}</p>
```

**Description** — `product.description`

**Farmer card:**
```tsx
<div className="bg-green-50 rounded-xl p-4 flex items-center gap-3">
  <div className="w-12 h-12 rounded-xl bg-green-700 flex items-center justify-center">
    <Sprout className="w-6 h-6 text-white" />
  </div>
  <div>
    <p className="font-bold text-[13px]">{product.farmer}</p>
    <p className="text-[11px] text-gray-500">{product.location}</p>
  </div>
  <Link to={`/financement`} className="ml-auto text-[11px] text-green-700 font-bold">
    {t('product.viewFarmer')} →
  </Link>
</div>
```

**Sticky CTA (bottom on mobile / right panel on desktop):**
```tsx
<button
  data-testid="invest-cta"
  onClick={() => setInvestOpen(true)}
  className="w-full h-12 bg-green-700 text-white rounded-xl font-display font-bold text-[15px]"
>
  {t('product.investCta')} — min ${product.minInvest}
</button>
```

`InvestModal` imported from S2-04 (prop-drilled `product`).

### Step 4 — i18n keys

```
product.roi          → "Rendement" / "Return"
product.duration     → "Durée" / "Duration"
product.minInvest    → "Min. investissement" / "Min. investment"
product.funded       → "financé" / "funded"
product.slotsLeft    → "places restantes" / "slots left"
product.viewFarmer   → "Voir l'agriculteur" / "View farmer"
product.investCta    → "Investir" / "Invest"
```

---

## Unit Tests — `src/pages/__tests__/ProductDetailScreen.test.tsx`

```typescript
vi.mock('@/hooks/useProducts', () => ({ useProduct: vi.fn() }))

it('renders product name and stats', () => {
  vi.mocked(useProduct).mockReturnValue({
    data: { id: 'p1', name: 'Pastèques', roi: 22, minInvest: 200, duration: 45, location: 'Songololo' },
    isLoading: false,
  } as any)
  render_()
  expect(screen.getByText('Pastèques')).toBeInTheDocument()
  expect(screen.getByText(/\+22%/)).toBeInTheDocument()
})

it('shows skeleton while loading', () => {
  vi.mocked(useProduct).mockReturnValue({ data: null, isLoading: true } as any)
  render_()
  expect(screen.getByRole('status')).toBeInTheDocument()
})

it('renders invest CTA button', () => {
  vi.mocked(useProduct).mockReturnValue({ data: { id: 'p1', name: 'P', minInvest: 200, roi: 22 }, isLoading: false } as any)
  render_()
  expect(screen.getByTestId('invest-cta')).toBeInTheDocument()
})
```

---

## ✅ Definition of Done
- [ ] `/market/:id` fetches product from Firestore by document ID
- [ ] Funding progress bar reflects real `invested` / `stock` values
- [ ] Farmer card rendered with product.farmer and product.location
- [ ] `data-testid="invest-cta"` on invest button
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s2-03): product detail screen wired to Firestore"
git push origin feature/s2-03-product-detail
```
