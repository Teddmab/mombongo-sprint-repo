# SM-03-01 — Course certificate unlock + preview

**Sprint:** SM-03 · Academia  
**Branch:** `feature/sm-03-academia`

## Context
`CertificatePreviewModal` component exists but is not triggered. A certificate should be issued when a user completes all modules in a course. The web app doesn't have this feature yet — mobile will lead here.

## Acceptance criteria
- [ ] When `completeModule` CF returns `courseCompleted: true`, `CertificatePreviewModal` opens automatically
- [ ] Certificate shows: user full name, course title, completion date, Mombongo logo
- [ ] "Télécharger" button generates a PNG via `react-native-view-shot` and opens the share sheet
- [ ] "Partager" button opens the native share sheet with the PNG
- [ ] Certificate is accessible from `CourseDetailScreen` via "Voir mon certificat" button if already completed
- [ ] Certificate data from: `httpsCallable(functions, "getCertificate")({ courseId })` returning `{ issuedAt, userName, courseTitle }`
- [ ] In devMode, returns mock certificate data

## Implementation notes
- `react-native-view-shot` for PNG export — add to package.json
- Certificate layout: white card with Mombongo green header, user name in large font, decorative border
- Store certificate issuance flag in Firestore via `completeModule` CF (already in SM-03-00)
