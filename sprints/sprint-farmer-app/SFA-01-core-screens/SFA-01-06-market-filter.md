# SFA-01-06 — Market Filter Sheet (Farmer App)

## Context
Port of SM-01-02. The web market screen has a filter sheet (crop type, province, price range, harvest date). The mobile `FarmerMarketScreen.tsx` (SFA-01-03) shows prices but has no filter. Farmers need to filter prices by province when they trade across regions.

## Scope
- Create `src/components/MarketFilterSheet.tsx` — bottom sheet with filter controls
- Add filter button to `FarmerMarketScreen.tsx` header
- Filters: Province (multi-select), Crop type (multi-select), Min price / Max price (CDF range slider)
- Filter state persists in component state (reset on sheet close if cancelled; applied on confirm)
- Filtered results passed back to `useMarketPrices` hook as params

## Files to create / modify
- `src/components/MarketFilterSheet.tsx`
- `src/screens/market/FarmerMarketScreen.tsx` — add filter button + apply filters

## Implementation

### MarketFilterSheet.tsx
```typescript
type FilterState = {
  provinces: string[]
  crops: string[]
  minPriceCdf: number | null
  maxPriceCdf: number | null
}

const ALL_PROVINCES = ['Kinshasa', 'Kongo Central', 'Kwilu', 'Kasaï', 'Maniema', 'Nord-Kivu', 'Sud-Kivu', 'Haut-Katanga', 'Lualaba']
const ALL_CROPS = ['Maïs', 'Manioc', 'Haricots', 'Riz', 'Arachides', 'Soja', 'Café', 'Cacao']

export function MarketFilterSheet({
  visible, onClose, onApply, initialFilters,
}: {
  visible: boolean
  onClose: () => void
  onApply: (filters: FilterState) => void
  initialFilters: FilterState
}) {
  const [filters, setFilters] = useState(initialFilters)

  return (
    <BottomSheet visible={visible} onClose={onClose}>
      <Text style={styles.title}>Filtrer les prix</Text>

      {/* Province multi-select chips */}
      <SectionLabel>Province</SectionLabel>
      <ChipGroup
        options={ALL_PROVINCES}
        selected={filters.provinces}
        onToggle={(p) => setFilters(f => ({
          ...f,
          provinces: f.provinces.includes(p) ? f.provinces.filter(x => x !== p) : [...f.provinces, p],
        }))}
      />

      {/* Crop type multi-select chips */}
      <SectionLabel>Culture</SectionLabel>
      <ChipGroup options={ALL_CROPS} selected={filters.crops} onToggle={...} />

      {/* Price range */}
      <SectionLabel>Prix (FC/kg)</SectionLabel>
      <PriceRangeInputs
        min={filters.minPriceCdf}
        max={filters.maxPriceCdf}
        onChange={(min, max) => setFilters(f => ({ ...f, minPriceCdf: min, maxPriceCdf: max }))}
      />

      <View style={styles.actions}>
        <Button variant="ghost" onPress={() => setFilters({ provinces: [], crops: [], minPriceCdf: null, maxPriceCdf: null })}>
          Réinitialiser
        </Button>
        <Button onPress={() => { onApply(filters); onClose() }}>
          Appliquer
        </Button>
      </View>
    </BottomSheet>
  )
}
```

### FarmerMarketScreen.tsx — filter integration
```typescript
const [filters, setFilters] = useState<FilterState>({ provinces: [], crops: [], minPriceCdf: null, maxPriceCdf: null })
const [filterVisible, setFilterVisible] = useState(false)

// Pass active filters to useMarketPrices:
const { data: prices } = useMarketPrices({
  provinces: filters.provinces.length ? filters.provinces : undefined,
  crops: filters.crops.length ? filters.crops : undefined,
})

// Client-side price range filter (applied after fetch):
const filteredPrices = prices?.filter(p =>
  (!filters.minPriceCdf || p.pricePerKgCdf >= filters.minPriceCdf) &&
  (!filters.maxPriceCdf || p.pricePerKgCdf <= filters.maxPriceCdf)
)

// Filter button in header with active-filter badge count:
<TouchableOpacity onPress={() => setFilterVisible(true)}>
  <FilterIcon />
  {activeFilterCount > 0 && <Badge count={activeFilterCount} />}
</TouchableOpacity>
```

## Acceptance criteria
- [ ] Filter button in market screen header (with badge showing active filter count)
- [ ] Province filter limits prices shown to selected provinces
- [ ] Crop filter limits to selected crop types
- [ ] Price range filter hides prices outside min/max range
- [ ] "Réinitialiser" clears all filters
- [ ] Filter persists while sheet is closed (stays applied until reset)

## Smoke test
1. Open Market tab → tap filter icon → sheet opens
2. Select "Kasaï" province → apply → confirm only Kasaï prices shown
3. Select "Maïs" crop → apply → confirm only maize price shown
4. Set min price 500 → apply → confirm prices below 500 hidden
5. Tap Réinitialiser → apply → confirm all prices return
6. Filter badge count matches number of active filter categories
