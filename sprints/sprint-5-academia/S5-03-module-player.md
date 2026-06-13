# S5-03 — Academia — Module Player (Video, PDF, Quiz)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S5-03 |
| Sprint | Sprint 5 — Academia |
| Branch | `feature/s5-03-module-player` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S5-02 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | /academia/:courseId/module/:moduleId — video/PDF/quiz player + progress tracking |
| `mombongo-functions` | 🔨 Active | markModuleComplete onCall |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### markModuleComplete onCall

Create `src/academia/markModuleComplete.ts`:

```typescript
export const markModuleComplete = functions.https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { courseId, moduleId }: { courseId: string; moduleId: string } = data

  const enrollmentSnap = await db.collection('enrollments')
    .where('userId', '==', uid)
    .where('courseId', '==', courseId)
    .limit(1)
    .get()

  if (enrollmentSnap.empty)
    throw new functions.https.HttpsError('not-found', 'Not enrolled in this course')

  const enrollDoc = enrollmentSnap.docs[0]
  const enrollment = enrollDoc.data()

  if (enrollment.completedModules.includes(moduleId))
    return { success: true }  // idempotent

  // Compute total module count for progress
  const modulesSnap = await db
    .collection('courses').doc(courseId)
    .collection('modules').get()
  const totalModules = modulesSnap.size

  const completed = [...enrollment.completedModules, moduleId]
  const progressPct = Math.round((completed.length / totalModules) * 100)
  const isCompleted = progressPct === 100

  const now = admin.firestore.FieldValue.serverTimestamp()
  await enrollDoc.ref.update({
    completedModules: admin.firestore.FieldValue.arrayUnion(moduleId),
    progressPct,
    ...(isCompleted ? { completedAt: now } : {}),
  })

  return { success: true, progressPct, isCompleted }
})
```

Export in `src/index.ts`.

---

## mombongo-web

### Step 1 — ModulePlayerScreen

Route: `/academia/:courseId/module/:moduleId`

```typescript
const { courseId, moduleId } = useParams<{ courseId: string; moduleId: string }>()
const { data: modules = [] } = useCourseModules(courseId!)
const module = modules.find(m => m.id === moduleId)
const { data: enrollment, refetch } = useMyEnrollment(courseId!)
const isCompleted = enrollment?.completedModules.includes(moduleId!) ?? false
const [quizAnswers, setQuizAnswers] = useState<Record<number, number>>({})
const [quizSubmitted, setQuizSubmitted] = useState(false)
```

**Video player** (when `module.type === 'video'`):
```tsx
<div className="aspect-video w-full rounded-2xl overflow-hidden bg-black">
  <iframe
    data-testid="youtube-player"
    src={`https://www.youtube.com/embed/${module.youtubeVideoId}?rel=0`}
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowFullScreen
    className="w-full h-full"
  />
</div>
```

**PDF viewer** (when `module.type === 'pdf'`):
```tsx
<div className="w-full rounded-2xl overflow-hidden border border-gray-200">
  <iframe
    data-testid="pdf-viewer"
    src={module.pdfUrl}
    className="w-full"
    style={{ height: '70vh' }}
    title={module.title}
  />
</div>
<a
  href={module.pdfUrl}
  target="_blank"
  rel="noopener noreferrer"
  className="mt-2 text-green-600 text-sm font-semibold flex items-center gap-1"
>
  📥 {t('academia.downloadPdf')}
</a>
```

**Quiz** (when `module.type === 'quiz'`):
```tsx
{module.questions?.map((q, qi) => (
  <div key={qi} className="mb-6">
    <p className="font-semibold text-[14px] mb-3">{q.q}</p>
    {q.options.map((opt, oi) => (
      <button
        key={oi}
        data-testid={`quiz-option-${qi}-${oi}`}
        onClick={() => !quizSubmitted && setQuizAnswers(prev => ({ ...prev, [qi]: oi }))}
        className={`w-full text-left p-3 rounded-xl mb-2 border transition
          ${quizSubmitted
            ? oi === q.answer ? 'border-green-500 bg-green-50' : 'border-gray-200 bg-gray-50'
            : quizAnswers[qi] === oi ? 'border-green-500 bg-green-50' : 'border-gray-200'
          }`}
      >
        {opt}
      </button>
    ))}
  </div>
))}
{!quizSubmitted && (
  <button
    data-testid="quiz-submit-btn"
    disabled={Object.keys(quizAnswers).length < (module.questions?.length ?? 0)}
    onClick={() => setQuizSubmitted(true)}
    className="w-full bg-green-600 text-white py-3 rounded-xl font-bold"
  >
    {t('academia.submitQuiz')}
  </button>
)}
{quizSubmitted && (
  <p data-testid="quiz-result" className="text-center font-bold text-green-700 mt-2">
    {t('academia.quizScore', {
      score: module.questions?.filter((q, i) => quizAnswers[i] === q.answer).length,
      total: module.questions?.length,
    })}
  </p>
)}
```

**Mark complete button** (shown for video/pdf types after content loads, or automatically after quiz submission):
```tsx
{!isCompleted && (module.type !== 'quiz' || quizSubmitted) && (
  <button
    data-testid="mark-complete-btn"
    onClick={handleMarkComplete}
    className="w-full bg-green-600 text-white py-4 font-bold rounded-2xl mt-4"
  >
    {t('academia.markComplete')}
  </button>
)}
{isCompleted && (
  <div data-testid="module-complete-badge" className="flex items-center gap-2 justify-center mt-4 text-green-700 font-bold">
    ✓ {t('academia.completed')}
  </div>
)}
```

```typescript
async function handleMarkComplete() {
  const result = await httpsCallable(functions, 'markModuleComplete')({
    courseId: courseId!, moduleId: moduleId!,
  }) as { data: { progressPct: number; isCompleted: boolean } }

  await refetch()   // refresh enrollment

  if (result.data.isCompleted) {
    // navigate to certificate / course completion screen
    navigate(`/academia/${courseId}/complete`)
  } else {
    // auto-advance to next module
    const currentIndex = modules.findIndex(m => m.id === moduleId)
    const nextModule = modules[currentIndex + 1]
    if (nextModule) navigate(`/academia/${courseId}/module/${nextModule.id}`)
  }
}
```

**Previous / Next navigation bar** at bottom:
```tsx
<div className="flex justify-between mt-6">
  {prevModule && (
    <button onClick={() => navigate(`/academia/${courseId}/module/${prevModule.id}`)}>
      ← {t('academia.prev')}
    </button>
  )}
  {nextModule && (
    <button onClick={() => navigate(`/academia/${courseId}/module/${nextModule.id}`)}>
      {t('academia.next')} →
    </button>
  )}
</div>
```

### Step 2 — Course Completion Screen

Route: `/academia/:courseId/complete`

```tsx
<div data-testid="course-complete-screen" className="text-center p-8">
  <span className="text-6xl">🎓</span>
  <h1 className="text-2xl font-extrabold mt-4">{t('academia.congratulations')}</h1>
  <p className="text-gray-500 mt-2">{t('academia.courseComplete', { title: course?.title })}</p>
  <button
    onClick={() => navigate('/academia')}
    className="mt-8 bg-green-600 text-white px-8 py-3 rounded-2xl font-bold"
  >
    {t('academia.backToCourses')}
  </button>
</div>
```

### Step 3 — i18n keys

```
academia.markComplete    → "Marquer comme terminé" / "Mark as complete"
academia.completed       → "Terminé" / "Completed"
academia.prev            → "Précédent" / "Previous"
academia.next            → "Suivant" / "Next"
academia.downloadPdf     → "Télécharger le PDF" / "Download PDF"
academia.submitQuiz      → "Soumettre" / "Submit quiz"
academia.quizScore       → "Score : {{score}}/{{total}}" / "Score: {{score}}/{{total}}"
academia.congratulations → "Félicitations !" / "Congratulations!"
academia.courseComplete  → "Vous avez terminé : {{title}}" / "You completed: {{title}}"
academia.backToCourses   → "Retour aux cours" / "Back to courses"
```

---

## ✅ Definition of Done
- [ ] Video modules render YouTube iframe
- [ ] PDF modules render iframe + download link
- [ ] Quiz shows all questions, validates on submit
- [ ] `markModuleComplete` updates `completedModules` and `progressPct` in enrollment
- [ ] Auto-advance to next module after marking complete
- [ ] Course completion screen shown at 100%
- [ ] `data-testid="mark-complete-btn"` present
- [ ] `npm run build` exits 0

```bash
firebase deploy --only functions:markModuleComplete,enrollCourse
git commit -m "feat(s5-03): module player — video/PDF/quiz + progress tracking"
git push origin feature/s5-03-module-player
```
