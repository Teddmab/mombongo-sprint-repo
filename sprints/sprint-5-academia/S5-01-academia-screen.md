# S5-01 — Academia — Course List Screen

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S5-01 |
| Sprint | Sprint 5 — Academia |
| Branch | `feature/s5-01-academia-screen` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S5-00 (courses seeded) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /academia — course cards from Firestore, featured section, category filter |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — useCourses hook

Create `src/hooks/useAcademia.ts`:

```typescript
import { useQuery } from '@tanstack/react-query'
import { collection, query, where, orderBy, getDocs } from 'firebase/firestore'
import { db, isDevMode } from '@/lib/firebase'
import { courses as MOCK_COURSES } from '@/data/mock'

export interface Course {
  id: string
  title: string
  titleEn: string
  description: string
  category: 'agriculture' | 'finance' | 'commerce' | 'technology'
  level: 'beginner' | 'intermediate' | 'advanced'
  durationMinutes: number
  moduleCount: number
  thumbnail: string
  instructor: string
  isFeatured: boolean
  enrollmentCount: number
  status: 'published' | 'draft'
}

export function useCourses(category?: string) {
  return useQuery({
    queryKey: ['courses', category],
    queryFn: async () => {
      if (isDevMode()) return MOCK_COURSES as unknown as Course[]
      const snap = await getDocs(
        query(
          collection(db, 'courses'),
          where('status', '==', 'published'),
          orderBy('createdAt', 'desc')
        )
      )
      let courses = snap.docs.map(d => ({ id: d.id, ...d.data() } as Course))
      if (category) courses = courses.filter(c => c.category === category)
      return courses
    },
    staleTime: 120_000,
  })
}

export function useFeaturedCourses() {
  const { data, ...rest } = useCourses()
  return { data: data?.filter(c => c.isFeatured) ?? [], ...rest }
}
```

Add mock data to `src/data/mock.ts`:
```typescript
export const courses: Partial<Course>[] = [
  {
    id: 'c1', title: 'Agriculture durable au Congo', category: 'agriculture',
    level: 'beginner', durationMinutes: 90, moduleCount: 4,
    instructor: 'Dr. Olivier Mwamba', isFeatured: true, enrollmentCount: 128,
    status: 'published', description: 'Maîtrisez les techniques modernes.',
  },
  {
    id: 'c2', title: 'Gestion financière pour agriculteurs', category: 'finance',
    level: 'intermediate', durationMinutes: 120, moduleCount: 5,
    instructor: 'Prof. Alice Kabila', isFeatured: true, enrollmentCount: 84,
    status: 'published', description: 'Accédez aux financements et planifiez.',
  },
  {
    id: 'c3', title: 'Commerce et export des produits', category: 'commerce',
    level: 'advanced', durationMinutes: 150, moduleCount: 6,
    instructor: 'Emmanuel Luyindula', isFeatured: false, enrollmentCount: 42,
    status: 'published', description: 'Vendez sur les marchés nationaux.',
  },
]
```

### Step 2 — Wire AcademiaScreen

In `src/pages/AcademiaScreen.tsx`, replace mock courses:

```typescript
const [categoryFilter, setCategoryFilter] = useState<string | undefined>()
const { data: courses = [], isLoading } = useCourses(categoryFilter)
const { data: featured = [] } = useFeaturedCourses()
```

**Category filter chips** (horizontal scroll):
- All / Agriculture / Finance / Commerce / Technologie
- Active chip: `bg-green-600 text-white`

**Featured section** (horizontal scroll of large cards, only shown when no filter active):
```tsx
{!categoryFilter && featured.length > 0 && (
  <section>
    <h2 className="font-bold text-[15px] mb-3">{t('academia.featured')}</h2>
    <div className="flex gap-3 overflow-x-auto pb-2 snap-x snap-mandatory">
      {featured.map(c => <FeaturedCourseCard key={c.id} course={c} />)}
    </div>
  </section>
)}
```

**FeaturedCourseCard** — wide card (w-72), thumbnail placeholder gradient, title, instructor, `{c.moduleCount} modules · {c.durationMinutes}min`, level badge.

**Course list grid** (2-column on mobile, 3-column on desktop):
Each card shows thumbnail, category chip, title, instructor, duration, level badge, enrollment count with person icon.

**Level badge colors**: `beginner` → green, `intermediate` → amber, `advanced` → red.

### Step 3 — i18n keys

```
academia.featured     → "À la une" / "Featured"
academia.allCourses   → "Tous les cours" / "All courses"
academia.filter       → "Catégorie" / "Category"
academia.modules      → "modules" / "modules"
academia.enrolled     → "inscrits" / "enrolled"
academia.beginner     → "Débutant" / "Beginner"
academia.intermediate → "Intermédiaire" / "Intermediate"
academia.advanced     → "Avancé" / "Advanced"
academia.empty        → "Aucun cours disponible" / "No courses available"
```

---

## Unit Tests — `src/pages/__tests__/AcademiaScreen.test.tsx`

```typescript
vi.mock('@/hooks/useAcademia', () => ({
  useCourses:        vi.fn(),
  useFeaturedCourses: vi.fn(),
}))

it('renders course cards', () => {
  vi.mocked(useCourses).mockReturnValue({
    data: [{ id: 'c1', title: 'Agriculture durable', category: 'agriculture',
              level: 'beginner', moduleCount: 4, durationMinutes: 90, enrollmentCount: 128 }],
    isLoading: false,
  } as any)
  vi.mocked(useFeaturedCourses).mockReturnValue({ data: [], isLoading: false } as any)
  render(<AcademiaScreen />)
  expect(screen.getByText(/Agriculture durable/)).toBeInTheDocument()
})
```

---

## ✅ Definition of Done
- [ ] Course list loads from Firestore `courses`
- [ ] Featured section shows `isFeatured: true` courses
- [ ] Category filter chips narrow the list client-side
- [ ] `data-testid="academia-screen"` on root element
- [ ] `npm run test:unit` passes
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s5-01): academia screen — course list from Firestore"
git push origin feature/s5-01-academia-screen
```
