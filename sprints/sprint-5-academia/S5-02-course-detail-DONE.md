# S5-02 — Academia — Course Detail and Enrollment

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S5-02 |
| Sprint | Sprint 5 — Academia |
| Branch | `feature/s5-02-course-detail` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S5-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /academia/:id — course detail, module list, enroll action |
| `mombongo-functions` | 🔨 Active | enrollCourse onCall |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### enrollCourse onCall

Create `src/academia/enrollCourse.ts`:

```typescript
export const enrollCourse = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { courseId }: { courseId: string } = data

  const courseSnap = await db.collection('courses').doc(courseId).get()
  if (!courseSnap.exists || courseSnap.data()?.status !== 'published')
    throw new functions.https.HttpsError('not-found', 'Course not available')

  // Idempotent: check existing enrollment
  const existing = await db.collection('enrollments')
    .where('userId', '==', uid)
    .where('courseId', '==', courseId)
    .limit(1)
    .get()

  if (!existing.empty) return { success: true, enrollmentId: existing.docs[0].id }

  const now = admin.firestore.FieldValue.serverTimestamp()
  const enrollRef = db.collection('enrollments').doc()

  await db.runTransaction(async tx => {
    tx.set(enrollRef, {
      userId: uid,
      courseId,
      completedModules: [],
      progressPct: 0,
      enrolledAt: now,
      completedAt: null,
    })
    tx.update(db.collection('courses').doc(courseId), {
      enrollmentCount: admin.firestore.FieldValue.increment(1),
    })
  })

  return { success: true, enrollmentId: enrollRef.id }
})
```

Export in `src/index.ts`.

---

## mombongo-web

### Step 1 — useCourse + useEnrollment hooks

Add to `src/hooks/useAcademia.ts`:

```typescript
// No Firestore SDK — all reads through Cloud Functions (db not exported from firebase.ts)
import { useAuth } from '@/hooks/useAuth'

export interface Module {
  id: string
  courseId: string
  order: number
  title: string
  type: 'video' | 'pdf' | 'quiz'
  youtubeVideoId?: string
  pdfUrl?: string
  durationMinutes: number
  isFree: boolean
  questions?: Array<{ q: string; options: string[]; answer: number }>
}

export interface Enrollment {
  id: string
  userId: string
  courseId: string
  completedModules: string[]
  progressPct: number
  enrolledAt: { seconds: number }
  completedAt: { seconds: number } | null
}

export function useCourse(id: string) {
  return useQuery({
    queryKey: ['course', id],
    queryFn: async () => {
      if (isDevMode()) return (MOCK_COURSES.find(c => c.id === id) ?? null) as Course | null
      const result = await httpsCallable<{ id: string }, { course: Course | null }>(functions, 'getCourse')({ id })
      return result.data.course
    },
    enabled: !!id,
  })
}

export function useCourseModules(courseId: string) {
  return useQuery({
    queryKey: ['modules', courseId],
    queryFn: async () => {
      if (isDevMode()) return MOCK_MODULES.filter(m => m.courseId === courseId) as Module[]
      const result = await httpsCallable<{ courseId: string }, { modules: Module[] }>(functions, 'getCourseModules')({ courseId })
      return result.data.modules
    },
    enabled: !!courseId,
  })
}

export function useMyEnrollment(courseId: string) {
  const { user } = useAuth()
  return useQuery({
    queryKey: ['enrollment', user?.uid, courseId],
    queryFn: async () => {
      if (!user?.uid || isDevMode()) return null
      const result = await httpsCallable<{ courseId: string }, { enrollment: Enrollment | null }>(functions, 'getMyEnrollment')({ courseId })
      return result.data.enrollment
    },
    enabled: !!user?.uid && !!courseId,
  })
}
```

> **`mombongo-functions` dependencies**: `getCourse` onCall (`{ id }` → `{ course }`), `getCourseModules` onCall (`{ courseId }` → modules subcollection ordered by `order` asc → `{ modules }`), `getMyEnrollment` onCall (`{ courseId }` → finds enrollment where `userId == context.auth.uid && courseId == courseId` → `{ enrollment }`).
```

Add `academiService` to service layer:
```typescript
enrollCourse: (payload: { courseId: string }) =>
  httpsCallable(functions, 'enrollCourse')(payload),
```

### Step 2 — Course detail screen

Route: `/academia/:id`

```typescript
const { id } = useParams<{ id: string }>()
const { data: course, isLoading } = useCourse(id!)
const { data: modules = [] } = useCourseModules(id!)
const { data: enrollment } = useMyEnrollment(id!)
const queryClient = useQueryClient()
```

Sections:
1. **Header** — thumbnail gradient, title, category chip + level badge, instructor name
2. **Stats row** — `{course.moduleCount} modules` / `{course.durationMinutes} min` / `{course.enrollmentCount} inscrits`
3. **Progress bar** (only if enrolled) — `enrollment.progressPct%`
4. **Module list** — each row shows: lock icon (if not `isFree` and not enrolled), `order.` title, type icon (🎬/📄/❓), duration
5. **Description** — `course.description`
6. **Sticky enroll / continue button**

**Enroll/Continue button logic:**
```typescript
const isEnrolled = !!enrollment
const nextModuleId = isEnrolled
  ? modules.find(m => !enrollment.completedModules.includes(m.id))?.id
  : null
```

```tsx
<button
  data-testid="course-enroll-btn"
  onClick={isEnrolled
    ? () => navigate(`/academia/${id}/module/${nextModuleId}`)
    : handleEnroll}
  className="w-full bg-green-600 text-white py-4 font-bold rounded-2xl"
>
  {isEnrolled ? t('academia.continue') : t('academia.enroll')}
</button>
```

```typescript
async function handleEnroll() {
  await academiService.enrollCourse({ courseId: id! })
  queryClient.invalidateQueries({ queryKey: ['enrollment'] })
  queryClient.invalidateQueries({ queryKey: ['course', id] })
}
```

**Module row locked state:**
```tsx
<div className={`flex items-center gap-3 p-3 rounded-xl mb-2
  ${!m.isFree && !isEnrolled ? 'opacity-50' : 'cursor-pointer hover:bg-gray-50'}
`}
  onClick={() => (m.isFree || isEnrolled) && navigate(`/academia/${id}/module/${m.id}`)}
>
  <span className="text-lg">{m.type === 'video' ? '🎬' : m.type === 'pdf' ? '📄' : '❓'}</span>
  <div className="flex-1">
    <p className="font-semibold text-[13px]">{m.title}</p>
    <p className="text-[11px] text-gray-500">{m.durationMinutes} min</p>
  </div>
  {!m.isFree && !isEnrolled && <span className="text-gray-400">🔒</span>}
  {enrollment?.completedModules.includes(m.id) && <span className="text-green-500">✓</span>}
</div>
```

### Step 3 — i18n keys

```
academia.enroll       → "S'inscrire gratuitement" / "Enroll for free"
academia.continue     → "Continuer" / "Continue"
academia.progress     → "Progression" / "Progress"
academia.locked       → "Module verrouillé — inscrivez-vous" / "Module locked — enroll first"
academia.modules_count → "{{n}} modules" / "{{n}} modules"
```

---

## ✅ Definition of Done
- [ ] `/academia/:id` loads course + modules from Firestore
- [ ] Module list shows lock icon for non-enrolled users
- [ ] `enrollCourse` creates enrollment doc and increments `enrollmentCount`
- [ ] Progress bar visible after enrollment
- [ ] `data-testid="course-enroll-btn"` present
- [ ] `npm run build` exits 0

```bash
firebase deploy --only functions:enrollCourse
git commit -m "feat(s5-02): course detail + enrollment flow + enrollCourse function"
git push origin feature/s5-02-course-detail
```
