# SFA-00-03 — Goal-First Onboarding (Farmer App)

## Context
Port of web sprint SU-01-03. The current mobile `OnboardingScreen.tsx` is a generic 4-slide carousel per role. For the Farmer App, we replace it with a 4-step goal-first flow:
1. **Votre première récolte** — select primary crop (maize, cassava, beans, rice, other)
2. **Votre province** — pick province from DRC province list
3. **Votre objectif** — select one goal: vendre ma récolte / obtenir un financement / suivre mes cultures / me former
4. **Recevoir les prix du marché** — push permission request with benefit framing

On completion, call `httpsCallable(functions, 'completeOnboarding')` with `{ cropType, province, goal }`. CF writes to `users/{uid}` (cropType, province, goal, onboardingComplete: true). Sends a welcome push.

The existing `OnboardingScreen.tsx` is replaced in the Farmer App only (guarded by `isFarmerApp`).

## Scope
- Create `src/screens/onboarding/FarmerOnboardingScreen.tsx` (4-step flow)
- Create `src/hooks/useCompleteOnboarding.ts` — calls `completeOnboarding` CF
- Guard: show `FarmerOnboardingScreen` if `isFarmerApp && !user.onboardingComplete`
- Push permission is requested at step 4 via `expo-notifications`
- On CF success, navigate to `FarmerHome`

## Cloud Function
`completeOnboarding` already planned in SU-01-03 for web. Same CF, same payload, reused on mobile.
```typescript
// CF signature (already planned — verify it exists)
httpsCallable(functions, 'completeOnboarding')({
  cropType: 'maize',
  province: 'Kinshasa',
  goal: 'sell_harvest',
})
```

## Files to create / modify
- `src/screens/onboarding/FarmerOnboardingScreen.tsx`
- `src/hooks/useCompleteOnboarding.ts`
- `src/navigation/RootNavigator.tsx` — show FarmerOnboarding before home if `!onboardingComplete`

## Implementation sketch

### FarmerOnboardingScreen.tsx
```typescript
const CROPS = ['Maïs', 'Manioc', 'Haricots', 'Riz', 'Autre']
const PROVINCES = ['Kinshasa', 'Kongo Central', 'Kwilu', 'Kasaï', 'Maniema', /* ... */]
const GOALS = [
  { key: 'sell_harvest', label: 'Vendre ma récolte' },
  { key: 'get_financing', label: 'Obtenir un financement' },
  { key: 'track_crops', label: 'Suivre mes cultures' },
  { key: 'learn', label: 'Me former' },
]

export function FarmerOnboardingScreen() {
  const [step, setStep] = useState(0)
  const [crop, setCrop] = useState('')
  const [province, setProvince] = useState('')
  const [goal, setGoal] = useState('')
  const { mutate: completeOnboarding, isPending } = useCompleteOnboarding()

  const handleFinish = async () => {
    // Step 4: request push permission before calling CF
    await requestPushPermission() // calls Notifications.requestPermissionsAsync()
    completeOnboarding({ cropType: crop, province, goal })
  }

  return (
    <SafeAreaView>
      <ProgressBar current={step} total={4} />
      {step === 0 && <CropPicker crops={CROPS} value={crop} onChange={setCrop} />}
      {step === 1 && <ProvincePicker provinces={PROVINCES} value={province} onChange={setProvince} />}
      {step === 2 && <GoalPicker goals={GOALS} value={goal} onChange={setGoal} />}
      {step === 3 && <PushPermissionStep onAllow={handleFinish} onSkip={handleFinish} loading={isPending} />}
      {step < 3 && (
        <Button onPress={() => setStep(s => s + 1)} disabled={!currentStepValid}>
          Suivant
        </Button>
      )}
    </SafeAreaView>
  )
}
```

### useCompleteOnboarding.ts
```typescript
import { useMutation } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'

export function useCompleteOnboarding() {
  return useMutation({
    mutationFn: (payload: { cropType: string; province: string; goal: string }) =>
      httpsCallable(functions, 'completeOnboarding')(payload),
  })
}
```

## Acceptance criteria
- [ ] New farmer user sees 4-step onboarding after first login
- [ ] Returning user with `onboardingComplete: true` skips directly to home
- [ ] Step 4 triggers `Notifications.requestPermissionsAsync()` before CF call
- [ ] On success, `users/{uid}` has `cropType`, `province`, `goal`, `onboardingComplete: true`
- [ ] Back button on step > 0 goes to previous step (not auth screen)
- [ ] "Skip" on push permission step still calls `completeOnboarding` (just without a token)

## Smoke test
1. Sign in as a new farmer user (fresh account, `onboardingComplete` not set)
2. Confirm 4-step flow appears
3. Complete all 4 steps — tap "Allow notifications" on step 4
4. Confirm navigation lands on FarmerHomeScreen
5. Reload app — confirm onboarding does not appear again
6. Open Firebase console → `users/{uid}` → verify `cropType`, `province`, `goal`, `onboardingComplete: true`
