# S2-02 — Marketplace — Filter Sheet & Search

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-02 |
| Sprint | Sprint 2 — Marketplace |
| Branch | `feature/s2-02-filter-sheet` |
| Merges into | `dev` |
| Estimate | 1.5h |
| Dependencies | S2-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | FilterSheet component, search input, client-side filter logic |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Current State
Product list loads from Firestore. No filtering or search.

### Step 1 — Filter state in MarketScreen

```typescript
// In MarketScreen component:
const [search, setSearch] = useState('')
const [filters, setFilters] = useState<{
  category: string | null
  minRoi: number | null
  maxDuration: number | null
}>({ category: null, minRoi: null, maxDuration: null })
const [filterOpen, setFilterOpen] = useState(false)

const { data: allProducts = [], isLoading } = useProducts()

const filtered = useMemo(() => {
  return allProducts.filter(p => {
    if (search && !p.name.toLowerCase().includes(search.toLowerCase())) return false
    if (filters.category && p.category !== filters.category) return false
    if (filters.minRoi && p.roi < filters.minRoi) return false
    if (filters.maxDuration && p.duration > filters.maxDuration) return false
    return true
  })
}, [allProducts, search, filters])
```

### Step 2 — Search bar UI

```tsx
<div className="relative">
  <Search className="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" />
  <input
    data-testid="market-search"
    value={search}
    onChange={e => setSearch(e.target.value)}
    placeholder={t('market.searchPlaceholder')}
    className="w-full h-10 pl-10 pr-4 bg-white border border-gray-200 rounded-xl text-[13px] focus:outline-none focus:ring-2 focus:ring-green-700/20"
  />
  {search && (
    <button onClick={() => setSearch('')} className="absolute right-3 top-1/2 -translate-y-1/2">
      <X className="w-3.5 h-3.5 text-gray-400" />
    </button>
  )}
</div>
```

### Step 3 — FilterSheet component

Create `src/components/FilterSheet.tsx`:

```typescript
interface FilterSheetProps {
  open: boolean
  onClose: () => void
  filters: { category: string | null; minRoi: number | null; maxDuration: number | null }
  onChange: (f: typeof filters) => void
  onReset: () => void
}
```

The sheet slides up from the bottom on mobile, appears as a sidebar panel on desktop. Contains:
- **Category chips**: Tous / Agriculture / Export / Logistique (maps to `"agriculture" | "export" | "logistique"`)
- **ROI minimum slider**: 0 – 40%, step 2
- **Duration max slider**: 0 – 120 days, step 15 (0 = all)
- **Reset** button clears all filters
- **Apply** button closes sheet and applies

Use `AnimatePresence + motion.div` from framer-motion for slide animation.

### Step 4 — Active filter chips

Show applied filters as dismissible chips below the search bar:
```tsx
{filters.category && (
  <span className="inline-flex items-center gap-1 text-[11px] bg-green-50 text-green-700 px-2 py-1 rounded-full font-bold">
    {t(`market.cat.${filters.category}`)}
    <button onClick={() => setFilters(f => ({ ...f, category: null }))}>
      <X className="w-3 h-3" />
    </button>
  </span>
)}
```

### Step 5 — i18n keys

```
market.searchPlaceholder  → "Rechercher un produit…" / "Search products…"
market.filter             → "Filtrer" / "Filter"
market.filterReset        → "Réinitialiser" / "Reset"
market.filterApply        → "Appliquer" / "Apply"
market.cat.agriculture    → "Agriculture" (same all langs)
market.cat.export         → "Export" (same all langs)
market.cat.logistique     → "Logistique" / "Logistics"
market.minRoi             → "ROI minimum" / "Min. ROI"
market.maxDuration        → "Durée maximale" / "Max duration"
```

---

## Unit Tests — `src/components/__tests__/FilterSheet.test.tsx`

```typescript
it('calls onChange when category chip clicked', () => {
  const onChange = vi.fn()
  render(<FilterSheet open={true} onClose={vi.fn()} filters={{...}} onChange={onChange} onReset={vi.fn()} />)
  fireEvent.click(screen.getByText(/Agriculture/i))
  expect(onChange).toHaveBeenCalledWith(expect.objectContaining({ category: 'agriculture' }))
})

it('calls onReset when reset button clicked', () => {
  const onReset = vi.fn()
  render(<FilterSheet open={true} ... onReset={onReset} />)
  fireEvent.click(screen.getByText(/Réinitialiser|Reset/i))
  expect(onReset).toHaveBeenCalled()
})
```

---

## ✅ Definition of Done
- [ ] Search input filters product list in real-time (client-side)
- [ ] FilterSheet opens/closes with animation
- [ ] Category, ROI, duration filters work correctly
- [ ] Active filters shown as dismissible chips
- [ ] `data-testid="market-search"` on search input
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s2-02): filter sheet + search for /market"
git push origin feature/s2-02-filter-sheet
```
