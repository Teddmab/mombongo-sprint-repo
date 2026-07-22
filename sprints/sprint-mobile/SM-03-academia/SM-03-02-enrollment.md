# SM-03-02 — Course enrollment + enrollment count

**Sprint:** SM-03 · Academia  
**Branch:** `feature/sm-03-academia`

## Context
`CourseDetailScreen` has an enroll CTA but enrollment may not be tracked in Firestore. Without enrollment, progress and certificates cannot be linked to the user.

## Acceptance criteria
- [ ] "S'inscrire" / "Commencer" CTA calls `httpsCallable(functions, "enrollCourse")({ courseId })`
- [ ] After enrollment, CTA changes to "Continuer" and module list becomes interactive
- [ ] `useEnrollment(courseId)` hook: returns `{ enrolled: boolean, progress: number, isLoading }`
- [ ] Enrollment state queried on `CourseDetailScreen` mount
- [ ] `enrollmentCount` on the course card increments after enrollment (via query invalidation)
- [ ] Free courses: enroll without subscription gate
- [ ] Premium courses (`course.isPremium === true`): check subscription before enrolling — if not subscribed, open `SubscriptionModal`
- [ ] In devMode, enrollment always succeeds; `enrolled` returns true

## Cloud Function
- `enrollCourse({ courseId })` — creates a `course_enrollments/{userId}_{courseId}` doc, increments `enrollmentCount` on the course

## Implementation notes
- `useEnrollment` calls `httpsCallable(functions, "getEnrollment")({ courseId })` on mount
- Only check subscription gate in production (not devMode)
