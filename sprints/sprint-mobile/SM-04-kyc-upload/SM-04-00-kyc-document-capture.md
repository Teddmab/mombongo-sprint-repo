# SM-04-00 — KYC document capture (camera / gallery)

**Sprint:** SM-04 · KYC Upload  
**Branch:** `feature/sm-04-kyc-upload`

## Context
The web app allows users to submit KYC documents (national ID, passport) but mobile has no document capture flow. Without KYC, users are blocked from investing above a threshold.

## Acceptance criteria
- [ ] `ProfileScreen` shows a "Vérifier mon identité" card with KYC status badge
- [ ] KYC card shows current status: none / pending / verified / rejected with reason
- [ ] "Commencer la vérification" button opens `KycUploadScreen` (new full-screen route)
- [ ] `KycUploadScreen` has 2 steps: (1) document type selection, (2) front + back photo capture
- [ ] Step 2: "Prendre une photo" opens camera via `expo-image-picker` with `launchCameraAsync`
- [ ] "Choisir depuis la galerie" alternative via `launchImageLibraryAsync`
- [ ] Selected images shown as previews with "Reprendre" option
- [ ] "Soumettre" button active only when both front and back are captured

## Implementation notes
- `expo-image-picker` is likely already installed (Expo SDK bundle)
- Request camera + media-library permissions before launch
- Image compression: `quality: 0.7`, `base64: false` (use URI for upload in SM-04-01)
- Route: `app/kyc.tsx` with `<Stack.Screen options={{ title: "Vérification KYC" }} />`
