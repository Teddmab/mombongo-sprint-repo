# SFA-04-01 — Academia Module Player + XP/Progress (Farmer App)

## Context
Web sprint SG-12 implemented Academia XP, progress tracking, and certificates. Mobile `AcademiaScreen.tsx` and `CourseDetailScreen.tsx` exist but use mock data. `ModulePlayerModal.tsx` exists but is not wired to real progress tracking CFs.

CFs already deployed (from SG-12):
- `getAcademiaModules` — list of modules with progress per farmer
- `completeModule` — marks a module completed, awards XP
- `getCertificate` — returns certificate data when course is 100% complete
- `enrollInCourse` — enroll in a course

## Scope
- Wire `AcademiaScreen.tsx` to `getAcademiaModules` CF
- Wire `ModulePlayerModal.tsx` to call `completeModule` CF on finish
- Show XP badge and level on AcademiaScreen
- Show course completion % progress bars
- Wire `CertificatePreviewModal.tsx` to real `getCertificate` CF

## Files to modify
- `src/hooks/useAcademia.ts` — convert to real CF calls
- `src/screens/AcademiaScreen.tsx` — show XP, progress bars
- `src/screens/CourseDetailScreen.tsx` — wire enrollment
- `src/components/ModulePlayerModal.tsx` — call completeModule on finish

## Implementation

### `src/hooks/useAcademia.ts`
```typescript
export function useAcademiaModules() {
  return useQuery({
    queryKey: ['academiaModules'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_MODULES
      const res = await httpsCallable<void, { modules: Module[] }>(
        functions, 'getAcademiaModules'
      )()
      return res.data.modules
    },
  })
}

export function useCompleteModule() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (payload: { moduleId: string; courseId: string }) =>
      httpsCallable(functions, 'completeModule')(payload),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['academiaModules'] })
      qc.invalidateQueries({ queryKey: ['academiaStreak'] })
    },
  })
}
```

### ModulePlayerModal.tsx — add completion call
```typescript
const { mutate: completeModule } = useCompleteModule()

// When video/content reaches 100%:
const handleModuleComplete = () => {
  completeModule({ moduleId, courseId })
  // Show XP earned toast: "+50 XP · Module terminé!"
  showToast(`+${module.xpReward} XP · Module terminé!`)
  onClose()
}
```

### AcademiaScreen — XP display
```typescript
// Above the course list:
<XpBadge xp={userData.academiaXp} level={userData.academiaLevel} />
// Each course card shows % completion progress bar
```

## Acceptance criteria
- [ ] Academia screen shows real modules from CF (not mock)
- [ ] Each course shows completion % progress bar
- [ ] XP badge shown at top of Academia screen
- [ ] Completing a module calls `completeModule` CF
- [ ] XP badge updates after module completion (without reload)
- [ ] Certificate modal shows real certificate when course is 100% complete
- [ ] Dev mode shows mock data

## Smoke test
1. Open Academia tab — confirm modules load from CF
2. Tap a course → enroll → open a module
3. Complete the module → confirm "+50 XP" toast appears
4. Return to Academia screen → confirm progress bar updated
5. Complete all modules in a course → open Certificate modal → confirm real certificate data
