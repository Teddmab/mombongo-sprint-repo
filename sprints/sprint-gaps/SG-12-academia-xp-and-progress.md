# SG-12 — Academia: Real XP, Progress & Certificates

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-12 |
| Sprint | Sprint Gaps 12 |
| Branch | `feature/sg-12-academia-progress` |
| Merges into | `dev` |
| Estimate | 5h |
| Dependencies | S5-01 through S5-04 (Academia screens built) |

---

## Context

The Academia screen header shows "0 XP · 0 cours terminés · 0 certificats" for all users — these
values are hardcoded to zero. The completion and course-detail flows (from Sprint 5) were wired
but the aggregated profile data was never saved back.

Also: there is no certificates screen — `CourseCompleteScreen` shows a congratulations page but
no downloadable certificate is generated.

---

## Firestore

### `user_progress/{userId}/courses/{courseId}`
Tracks per-course progress (already exists from S5 — reviewed to make sure it's being written):
```
{
  courseId: string
  status: 'not_started' | 'in_progress' | 'completed'
  completedModules: string[]     // module IDs
  xpEarned: number
  completedAt?: Timestamp
  certificateId?: string
  lastActivityAt: Timestamp
}
```

### `certificates/{id}`
New collection for awarded certificates:
```
{
  userId: string
  courseId: string
  courseTitle: string
  xpEarned: number
  issuedAt: Timestamp
  certificateNumber: string      // CERT-2026-00042
}
```

---

## Cloud Functions

### `getAcademiaProfile`
```typescript
// Returns aggregated XP, completions, certificates count
export const getAcademiaProfile = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const progressSnap = await db.collection('user_progress').doc(uid)
    .collection('courses').where('status', '==', 'completed').get()

  const totalXp = progressSnap.docs.reduce((sum: number, d: any) => sum + (d.data().xpEarned ?? 0), 0)
  const completedCount = progressSnap.size

  const certsSnap = await db.collection('certificates').where('userId', '==', uid).get()

  return {
    totalXp,
    completedCourses: completedCount,
    certificates: certsSnap.size,
  }
})
```

### `getMyCertificates`
```typescript
export const getMyCertificates = functions.region('europe-west1').https.onCall(async (_, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const snap = await db.collection('certificates')
    .where('userId', '==', uid)
    .orderBy('issuedAt', 'desc')
    .get()

  return { certificates: snap.docs.map(d => ({ id: d.id, ...d.data() })) }
})
```

### Update `completeCourse` CF (or `CourseCompleteScreen` flow)
When a user completes a course, issue a certificate:
```typescript
// Inside the course-completion CF (or at the end of completeCourse):
const countSnap = await db.collection('certificates').count().get()
const certificateNumber = `CERT-${new Date().getFullYear()}-${String(countSnap.data().count + 1).padStart(5, '0')}`

await db.collection('certificates').add({
  userId: uid,
  courseId,
  courseTitle: course.title,
  xpEarned: course.xpReward ?? 50,
  issuedAt: FieldValue.serverTimestamp(),
  certificateNumber,
})

// Update user_progress
await db.collection('user_progress').doc(uid).collection('courses').doc(courseId).set({
  status: 'completed',
  xpEarned: course.xpReward ?? 50,
  completedAt: FieldValue.serverTimestamp(),
  certificateId: certRef.id,
}, { merge: true })
```

---

## Web Hook

`src/hooks/useAcademia.ts` — add or update:
```typescript
export function useAcademiaProfile() {
  if (isDevMode()) return { data: { totalXp: 120, completedCourses: 2, certificates: 1 }, isLoading: false }
  return useQuery({
    queryKey: ['academia-profile'],
    queryFn: async () => {
      const res = await httpsCallable(functions, 'getAcademiaProfile')({})
      return res.data as AcademiaProfile
    },
    staleTime: 120_000,
  })
}

export function useMyCertificates() {
  if (isDevMode()) return { data: { certificates: MOCK_CERTIFICATES }, isLoading: false }
  return useQuery({
    queryKey: ['my-certificates'],
    queryFn: async () => {
      const res = await httpsCallable(functions, 'getMyCertificates')({})
      return res.data as { certificates: Certificate[] }
    },
  })
}
```

---

## UI Changes

### `AcademiaScreen.tsx` — XP header strip
Replace hardcoded "0 XP · 0 cours terminés · 0 certificats" with real data:
```tsx
const { data: profile } = useAcademiaProfile()

// Header:
<span>{profile?.totalXp ?? 0} XP</span>
<span>{profile?.completedCourses ?? 0} cours terminés</span>
<span>{profile?.certificates ?? 0} certificats</span>
```

### `CourseCompleteScreen.tsx` — show certificate number
After completing a course, the completion screen displays:
```
🎓 Félicitations !
Vous avez terminé "Introduction à l'investissement agricole"
+50 XP gagné · Certificat CERT-2026-00001 émis

[Voir mon certificat →]   [Continuer les cours]
```

"Voir mon certificat" navigates to `/academia/certificates`.

### New route: `/academia/certificates`
A simple list of issued certificates:
```
Mes certificats

  ┌────────────────────────────────────────────┐
  │  📜 CERT-2026-00001                        │
  │  Introduction à l'investissement agricole  │
  │  Émis le 15 juil. 2026 · 50 XP            │
  │  [Télécharger PDF]  (future — toast stub)  │
  └────────────────────────────────────────────┘
```

"Télécharger PDF" is a toast stub ("Bientôt disponible") for now — PDF generation is S12-xx scope.

---

## Acceptance Criteria
- [ ] `getAcademiaProfile` CF returns real XP, completed courses, certificates count
- [ ] `getMyCertificates` CF returns list of issued certificates
- [ ] `completeCourse` flow issues a certificate doc with auto-generated number
- [ ] `AcademiaScreen` header shows real XP / completed / certificates (not zeros)
- [ ] `CourseCompleteScreen` shows certificate number after completion
- [ ] `/academia/certificates` route exists and lists user's certificates
- [ ] "Télécharger PDF" is a toast stub (no PDF generation yet)
- [ ] Dev mode: `useAcademiaProfile` returns mock values (120 XP, 2 courses, 1 cert)
