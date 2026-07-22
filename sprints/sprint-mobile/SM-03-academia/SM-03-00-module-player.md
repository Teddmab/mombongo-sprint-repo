# SM-03-00 — Module player: real data wiring + progress tracking

**Sprint:** SM-03 · Academia  
**Branch:** `feature/sm-03-academia`

## Context
`ModulePlayerModal` component exists but needs to be wired to real module data (video URL, PDF URL, or quiz questions) and must track completion. Currently the modal opens but progress is not persisted to Firestore via CF.

## Acceptance criteria
- [ ] `CourseDetailScreen` opens `ModulePlayerModal` with real module data
- [ ] `ModulePlayerModal` handles all 3 module types:
  - `video`: renders a `<Video>` via `expo-av` with play/pause, progress bar
  - `pdf`: renders a WebView with the signed PDF URL (from CF `getModuleAsset`)
  - `quiz`: renders questions one by one with multi-choice, shows score at end
- [ ] On completion (video watched 90%+, PDF scrolled to end, quiz submitted), calls `httpsCallable(functions, "completeModule")` with `{ courseId, moduleId }`
- [ ] CF response `{ completedCount, totalModules, courseCompleted }` updates local state
- [ ] Completed modules show a checkmark in the module list
- [ ] Progress bar on the course card reflects `completedCount / totalModules`
- [ ] In devMode, `completeModule` is skipped; module marked complete locally

## Data shape
```ts
interface ModuleAsset {
  videoUrl?: string;   // signed GCS URL for video
  pdfUrl?: string;     // signed GCS URL for PDF
  quiz?: QuizQuestion[];
}
interface QuizQuestion {
  q: string;
  choices: string[];
  correct: number; // index
}
```

## Implementation notes
- `expo-av` is not yet installed — add to package.json
- PDF: use a `WebView` (react-native-webview) pointing to the signed PDF URL
- Quiz: store answers in local state; submit at the end
- Progress persistence: `completedModuleIds` stored in user's Firestore profile via CF
