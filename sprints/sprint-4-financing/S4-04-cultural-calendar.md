# S4-04 — Financing — Cultural Calendar

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S4-04 |
| Sprint | Sprint 4 — Financing |
| Branch | `feature/s4-04-cultural-calendar` |
| Merges into | `dev` |
| Estimate | 1.5h |
| Dependencies | S4-00 (cultural_events seeded), S4-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Cultural calendar tab inside FinancingScreen |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — useCulturalEvents hook

Add to `src/hooks/useFinancing.ts`:

```typescript
export interface CulturalEvent {
  id: string
  cropType: string
  eventType: 'planting' | 'harvest' | 'fertilizing' | 'irrigation'
  monthStart: number   // 1–12
  monthEnd: number
  description: string
}

export function useCulturalEvents(cropType?: string) {
  return useQuery({
    queryKey: ['cultural-events', cropType],
    queryFn: async () => {
      if (isDevMode()) return MOCK_CULTURAL_EVENTS as unknown as CulturalEvent[]
      const snap = await getDocs(collection(db, 'cultural_events'))
      let events = snap.docs.map(d => ({ id: d.id, ...d.data() } as CulturalEvent))
      if (cropType) events = events.filter(e => e.cropType === cropType)
      return events.sort((a, b) => a.monthStart - b.monthStart)
    },
    staleTime: 3_600_000,  // calendar data rarely changes
  })
}
```

Add mock data to `src/data/mock.ts`:
```typescript
export const MOCK_CULTURAL_EVENTS: Partial<CulturalEvent>[] = [
  { id: 'ce1', cropType: 'Maïs',   eventType: 'planting', monthStart: 10, monthEnd: 11, description: 'Semis de maïs' },
  { id: 'ce2', cropType: 'Maïs',   eventType: 'harvest',  monthStart: 3,  monthEnd: 4,  description: 'Récolte maïs' },
  { id: 'ce3', cropType: 'Manioc', eventType: 'planting', monthStart: 9,  monthEnd: 10, description: 'Bouturage manioc' },
  { id: 'ce4', cropType: 'Manioc', eventType: 'harvest',  monthStart: 9,  monthEnd: 12, description: 'Récolte manioc' },
  { id: 'ce5', cropType: 'Soja',   eventType: 'planting', monthStart: 4,  monthEnd: 5,  description: 'Semis soja' },
  { id: 'ce6', cropType: 'Soja',   eventType: 'harvest',  monthStart: 8,  monthEnd: 9,  description: 'Récolte soja' },
  { id: 'ce7', cropType: 'Cacao',  eventType: 'harvest',  monthStart: 10, monthEnd: 2,  description: 'Grande récolte cacao' },
]
```

### Step 2 — Calendar UI

Add a "Calendrier" tab to `FinancingScreen` (alongside the farmer list tab).

```tsx
const MONTH_LABELS = ['Jan','Fév','Mar','Avr','Mai','Jun','Jul','Aoû','Sep','Oct','Nov','Déc']
const EVENT_COLORS = {
  planting:    'bg-green-100 border-green-400 text-green-800',
  harvest:     'bg-amber-100 border-amber-400 text-amber-800',
  fertilizing: 'bg-blue-100  border-blue-400  text-blue-800',
  irrigation:  'bg-sky-100   border-sky-400   text-sky-800',
}

// Group events by cropType
const grouped = events.reduce<Record<string, CulturalEvent[]>>((acc, e) => {
  acc[e.cropType] = [...(acc[e.cropType] ?? []), e]
  return acc
}, {})
```

Layout — one row per crop type, 12 columns (months):

```tsx
<div className="overflow-x-auto" data-testid="cultural-calendar">
  {/* Header row — month labels */}
  <div className="grid grid-cols-13 text-xs text-gray-400 mb-2">
    <div className="font-semibold text-gray-700">{t('calendar.crop')}</div>
    {MONTH_LABELS.map(m => <div key={m} className="text-center">{m}</div>)}
  </div>

  {Object.entries(grouped).map(([crop, cropEvents]) => (
    <div key={crop} className="grid grid-cols-13 items-center mb-3">
      <div className="text-sm font-semibold text-gray-700 pr-2">{crop}</div>
      {Array.from({ length: 12 }, (_, i) => {
        const month = i + 1
        const event = cropEvents.find(e =>
          month >= e.monthStart && month <= e.monthEnd
        )
        return (
          <div
            key={i}
            className={`h-7 rounded mx-0.5 border text-[10px] flex items-center justify-center
              ${event ? EVENT_COLORS[event.eventType] : 'bg-gray-50 border-gray-100'}`}
            title={event?.description}
          >
            {event ? (event.eventType === 'planting' ? '🌱' : '🌾') : ''}
          </div>
        )
      })}
    </div>
  ))}
</div>
```

**Legend row:**
```tsx
<div className="flex gap-3 mt-4 text-xs">
  <span className="flex items-center gap-1"><span className="w-3 h-3 rounded bg-green-300" /> {t('calendar.planting')}</span>
  <span className="flex items-center gap-1"><span className="w-3 h-3 rounded bg-amber-300" /> {t('calendar.harvest')}</span>
</div>
```

**Today indicator** — highlight current month column with a subtle ring:
```typescript
const currentMonth = new Date().getMonth() + 1   // 1-indexed
```

### Step 3 — i18n keys

```
calendar.title      → "Calendrier cultural" / "Cultural calendar"
calendar.crop       → "Culture" / "Crop"
calendar.planting   → "Semis" / "Planting"
calendar.harvest    → "Récolte" / "Harvest"
calendar.today      → "Ce mois" / "This month"
```

---

## Unit Tests — `src/hooks/__tests__/useFinancing.test.ts`

```typescript
import { useCulturalEvents } from '@/hooks/useFinancing'

describe('useCulturalEvents mock', () => {
  it('returns events sorted by monthStart', async () => {
    vi.mock('@/lib/firebase', () => ({ db: {}, isDevMode: vi.fn(() => true) }))
    const { result } = renderHook(() => useCulturalEvents())
    await waitFor(() => expect(result.current.data).toBeDefined())
    const months = result.current.data!.map(e => e.monthStart)
    expect(months).toEqual([...months].sort((a, b) => a - b))
  })
})
```

---

## ✅ Definition of Done
- [ ] Calendar renders all crop types as rows with 12 month columns
- [ ] Planting and harvest events shown with color coding
- [ ] Current month column highlighted
- [ ] `data-testid="cultural-calendar"` on calendar root
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s4-04): cultural calendar — monthly crop event grid"
git push origin feature/s4-04-cultural-calendar
```
