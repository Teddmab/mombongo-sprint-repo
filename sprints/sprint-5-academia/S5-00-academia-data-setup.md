# S5-00 — Academia — Data Setup: Courses, Modules & Admin CRUD

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S5-00 |
| Sprint | Sprint 5 — Academia |
| Branch | `feature/s5-00-academia-data-setup` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S2-00 (functions project initialized) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | 🔨 Active | Seed courses + modules collections; Firestore rules |
| `mombongo-admin` | 🔨 Active | AdminAcademia screen — manage courses + modules |
| `mombongo-web` | ✅ Done | — |

---

## mombongo-functions

### Step 1 — Seed courses and modules

Create `src/scripts/seedAcademia.ts`:

```typescript
import { db } from '../lib/admin'
import { FieldValue } from 'firebase-admin/firestore'

const courses = [
  {
    title: 'Agriculture durable au Congo',
    titleEn: 'Sustainable Agriculture in Congo',
    description: 'Maîtrisez les techniques modernes pour maximiser vos rendements de façon écologique.',
    category: 'agriculture',
    level: 'beginner',
    durationMinutes: 90,
    moduleCount: 4,
    thumbnail: '',
    instructor: 'Dr. Olivier Mwamba',
    isFeatured: true,
    enrollmentCount: 0,
    status: 'published',
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    title: 'Gestion financière pour agriculteurs',
    titleEn: 'Financial Management for Farmers',
    description: 'Apprenez à gérer votre budget, accéder aux financements et planifier votre saison.',
    category: 'finance',
    level: 'intermediate',
    durationMinutes: 120,
    moduleCount: 5,
    thumbnail: '',
    instructor: 'Prof. Alice Kabila',
    isFeatured: true,
    enrollmentCount: 0,
    status: 'published',
    createdAt: FieldValue.serverTimestamp(),
  },
  {
    title: 'Commerce et export des produits agricoles',
    titleEn: 'Trade and Export of Agricultural Products',
    description: 'Comment vendre vos produits sur les marchés nationaux et internationaux.',
    category: 'commerce',
    level: 'advanced',
    durationMinutes: 150,
    moduleCount: 6,
    thumbnail: '',
    instructor: 'Emmanuel Luyindula',
    isFeatured: false,
    enrollmentCount: 0,
    status: 'published',
    createdAt: FieldValue.serverTimestamp(),
  },
]

// modules for course 1
const modules_course1 = [
  {
    courseId: 'PLACEHOLDER',
    order: 1,
    title: 'Introduction à l\'agriculture régénérative',
    type: 'video',              // video | pdf | quiz
    youtubeVideoId: 'dQw4w9WgXcQ',   // placeholder
    durationMinutes: 18,
    isFree: true,
  },
  {
    courseId: 'PLACEHOLDER',
    order: 2,
    title: 'Gestion de l\'eau et irrigation',
    type: 'video',
    youtubeVideoId: 'dQw4w9WgXcQ',
    durationMinutes: 22,
    isFree: false,
  },
  {
    courseId: 'PLACEHOLDER',
    order: 3,
    title: 'Guide de compostage — PDF',
    type: 'pdf',
    pdfUrl: '',   // Storage URL set after upload
    durationMinutes: 15,
    isFree: false,
  },
  {
    courseId: 'PLACEHOLDER',
    order: 4,
    title: 'Quiz — Agriculture durable',
    type: 'quiz',
    questions: [
      { q: 'Quel engrais naturel améliore la structure du sol ?', options: ['Urée','Compost','DAP','NPK'], answer: 1 },
    ],
    durationMinutes: 10,
    isFree: false,
  },
]

async function seed() {
  for (const course of courses) {
    const ref = await db.collection('courses').add(course)
    // modules belong to first course only in this seed
    if (course.title.startsWith('Agriculture durable')) {
      for (const mod of modules_course1) {
        await db.collection('courses').doc(ref.id).collection('modules').add({
          ...mod,
          courseId: ref.id,
        })
      }
    }
  }
  console.log('Academia seeded')
}
seed().catch(console.error)
```

### Step 2 — Firestore security rules

```
match /courses/{courseId} {
  allow read: if true;
  allow write: if isAdmin();

  match /modules/{moduleId} {
    allow read: if request.auth != null;
    allow write: if isAdmin();
  }
}

match /enrollments/{enrollmentId} {
  allow read:   if request.auth != null
    && resource.data.userId == request.auth.uid;
  allow create: if request.auth != null
    && request.resource.data.userId == request.auth.uid;
  allow update: if request.auth != null
    && resource.data.userId == request.auth.uid;  // progress updates
}
```

---

## mombongo-admin

### AdminAcademia screen

`src/pages/AdminAcademia.tsx` — two tabs:

**Tab 1 — Cours**: table with title / category / level / moduleCount / status / enrollmentCount. Actions: create, edit, toggle status (draft → published), delete.

Create/edit uses a drawer form:
- `title`, `titleEn`, `description`, `category` (select), `level` (select), `instructor`
- `status` toggle
- After save → refresh query

**Tab 2 — Modules**: select a course from dropdown, list its modules ordered by `order`. Actions: add module, reorder (drag-and-drop or up/down buttons), edit title + type + youtubeVideoId/pdfUrl, delete.

```typescript
// Queries
const coursesQ = useQuery({
  queryKey: ['admin-courses'],
  queryFn: () => getDocs(query(collection(db, 'courses'), orderBy('createdAt', 'desc'))),
})

const modulesQ = useQuery({
  queryKey: ['admin-modules', selectedCourseId],
  queryFn: () => selectedCourseId
    ? getDocs(query(collection(db, 'courses', selectedCourseId, 'modules'), orderBy('order', 'asc')))
    : Promise.resolve(null),
  enabled: !!selectedCourseId,
})
```

Route: add `/academia` to admin router.

---

## ✅ Definition of Done
- [ ] Seed script populates `courses` and subcollection `courses/{id}/modules`
- [ ] Admin `/academia` lists courses, allows create/edit/status toggle
- [ ] Admin can add and reorder modules for a course
- [ ] Firestore rules deployed
- [ ] `npm run build` exits 0 (admin)

```bash
git commit -m "feat(s5-00): academia data setup — seed script + admin CRUD"
git push origin feature/s5-00-academia-data-setup
```
