# SFA-00-05 — Profile Management (Farmer App)

## Context
Farmers need to edit their profile (name, phone, avatar), change their password, select app language (FR/SW/LN), and view their account status (KYC level, wallet balance). The web has ProfileScreen with these features wired to CFs. Mobile has `ProfileScreen.tsx` already but it largely mocks data or calls stubs.

CFs already deployed:
- `updateProfile` — updates displayName, phone, province, cropType
- `changePassword` — Firebase Auth `updatePassword` (client-side, no CF needed)
- `getWalletBalance` — returns current wallet balance in CDF

Photo upload: same signed URL pattern as KYC (`getProfilePhotoUploadUrl` CF → PUT → `confirmProfilePhoto` CF).

## Scope
- Wire `ProfileScreen.tsx` to `updateProfile` CF
- Add editable fields: display name, phone number, province, primary crop
- Add avatar upload via camera or gallery (signed URL pattern)
- Add language picker (French / Swahili / Lingala) — saves to AsyncStorage + i18n
- Show wallet balance from `getWalletBalance` CF
- Add KYC status chip linking to `KycUploadScreen` (SFA-03-02)
- Change password via Firebase Auth client SDK (not a CF — Auth is allowed from frontend)

## Files to modify
- `src/screens/ProfileScreen.tsx` — wire all fields
- `src/hooks/useProfile.ts` — new, wraps `updateProfile` CF

## Implementation

### `src/hooks/useProfile.ts`
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { updatePassword } from 'firebase/auth'
import { auth, functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

export function useUpdateProfile() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (payload: { displayName?: string; phone?: string; province?: string; cropType?: string }) =>
      httpsCallable(functions, 'updateProfile')(payload),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['userProfile'] }),
  })
}

export function useChangePassword() {
  return useMutation({
    mutationFn: (newPassword: string) => {
      const user = auth.currentUser
      if (!user) throw new Error('Not authenticated')
      return updatePassword(user, newPassword)
    },
  })
}

export function useWalletBalance() {
  return useQuery({
    queryKey: ['walletBalance'],
    queryFn: async () => {
      if (isDevMode()) return { balanceCdf: 125_000 }
      const res = await httpsCallable<void, { balanceCdf: number }>(functions, 'getWalletBalance')()
      return res.data
    },
    staleTime: 5 * 60_000,
  })
}
```

### Avatar upload helper (reuse KYC pattern)
```typescript
const uploadAvatar = async (asset: ImagePickerAsset): Promise<void> => {
  const { data } = await httpsCallable<{ type: 'avatar' }, { uploadUrl: string }>(
    functions, 'getProfilePhotoUploadUrl'
  )({ type: 'avatar' })
  const blob = await (await fetch(asset.uri)).blob()
  await fetch(data.uploadUrl, { method: 'PUT', body: blob, headers: { 'Content-Type': 'image/jpeg' } })
  await httpsCallable(functions, 'confirmProfilePhoto')({})
}
```

### Language selection
```typescript
import * as Localization from 'expo-localization'
import AsyncStorage from '@react-native-async-storage/async-storage'
import i18n from '@/lib/i18n'

const LANGUAGES = [
  { code: 'fr', label: 'Français' },
  { code: 'sw', label: 'Kiswahili' },
  { code: 'ln', label: 'Lingala' },
]

const changeLanguage = async (code: string) => {
  await AsyncStorage.setItem('app_language', code)
  i18n.changeLanguage(code)
}
```

### ProfileScreen.tsx layout
```typescript
// Avatar with edit overlay (tap → image picker)
// Name field (editable)
// Phone field (editable)
// Province + crop type fields (editable)
// Wallet balance row with "Recharger" and "Retirer" buttons
// KYC status chip (tap → KycUploadScreen)
// Language picker
// "Changer mot de passe" row → ChangePasswordSheet
// "Se déconnecter" button
```

## Acceptance criteria
- [ ] Editing name/phone/province/cropType calls `updateProfile` CF
- [ ] Avatar tap opens image picker; selected photo uploads via signed URL
- [ ] Wallet balance shown from `getWalletBalance` CF
- [ ] Language switch persists across app restarts
- [ ] Change password works without calling a CF (Firebase Auth client SDK)
- [ ] KYC chip shows correct status and taps to KycUploadScreen

## Smoke test
1. Open Profile → edit display name → save → verify change persists on reload
2. Tap avatar → pick a photo → verify avatar updates
3. Change language to Kiswahili → kill app → reopen → verify language retained
4. Tap "Changer mot de passe" → enter new password → verify login works with new password
5. Wallet balance shows non-zero value in live mode
