# S3-02 — Bourse — Opportunity Detail

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S3-02 |
| Sprint | Sprint 3 — Bourse |
| Branch | `feature/s3-02-bourse-detail` |
| Merges into | `dev` |
| Estimate | 1.5h |
| Dependencies | S3-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /bourse/:id — opportunity detail with price history chart |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — useBourseOpportunity + usePriceHistory hooks

Add to `src/hooks/useBourse.ts`:

```typescript
import { doc, getDoc, query, collection, where, orderBy, limit, getDocs } from 'firebase/firestore'

export function useBourseOpportunity(id: string) {
  return useQuery({
    queryKey: ['bourse-opportunity', id],
    queryFn: async () => {
      if (isDevMode()) return MOCK_OPPS.find(o => o.id === id) ?? null
      const snap = await getDoc(doc(db, 'bourse_opportunities', id))
      return snap.exists() ? ({ id: snap.id, ...snap.data() } as BourseOpportunity) : null
    },
    enabled: !!id,
  })
}

export function usePriceHistory(commodity: string) {
  return useQuery({
    queryKey: ['bourse-prices', commodity],
    queryFn: async () => {
      if (isDevMode()) return []
      const snap = await getDocs(
        query(
          collection(db, 'bourse_prices'),
          where('productName', '==', commodity),
          orderBy('recordedAt', 'desc'),
          limit(30)
        )
      )
      return snap.docs.map(d => d.data() as BoursePrice).reverse()
    },
    enabled: !!commodity,
    staleTime: 300_000,
  })
}
```

### Step 2 — Screen layout

Route: `/bourse/:id`

```typescript
const { id } = useParams<{ id: string }>()
const { data: opp, isLoading } = useBourseOpportunity(id!)
const { data: history = [] } = usePriceHistory(opp?.commodity ?? '')
```

Sections:
1. **Header** — route (large), commodity name + status chip, departure date
2. **Stats row** — ROI / duration / min invest / capacity
3. **Funding progress** — `filledKg / capacityKg` bar + investors count
4. **Price history chart** — recharts AreaChart of `history.map(p => ({ v: p.priceCdfPerKg }))` (same pattern as HomeScreen portfolio chart)
5. **Description** — from `opp.description`
6. **Sticky invest button** — `data-testid="bourse-invest-cta"`, opens BourseInvestModal (S3-03)

### Step 3 — i18n keys

```
bourse.priceHistory  → "Historique des prix" / "Price history"
bourse.investors     → "investisseurs" / "investors"
bourse.departsOn     → "Départ le" / "Departs on"
bourse.duration      → "Transit" / "Transit"
```

---

## ✅ Definition of Done
- [ ] `/bourse/:id` loads opportunity from Firestore
- [ ] Price history chart renders (empty if no history)
- [ ] Capacity progress bar reflects real `filledKg / capacityKg`
- [ ] `data-testid="bourse-invest-cta"` present
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s3-02): bourse detail screen with price history"
git push origin feature/s3-02-bourse-detail
```
