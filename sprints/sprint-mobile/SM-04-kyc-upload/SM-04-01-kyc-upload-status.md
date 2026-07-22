# SM-04-01 — KYC upload to signed URL + status polling

**Sprint:** SM-04 · KYC Upload  
**Branch:** `feature/sm-04-kyc-upload`

## Context
After capturing documents (SM-04-00), images must be uploaded to Firebase Storage via a signed URL provided by a Cloud Function (never directly from frontend). Then the KYC record must be created and status polled.

## Acceptance criteria
- [ ] "Soumettre" calls `httpsCallable(functions, "getKycUploadUrls")` with `{ docType, fileTypes: ["front", "back"] }` → returns `{ frontUploadUrl, backUploadUrl }`
- [ ] Images uploaded via `fetch(signedUrl, { method: "PUT", body: fileBlob })` — no Firebase Storage SDK
- [ ] After upload, calls `httpsCallable(functions, "submitKyc")({ docType, frontPath, backPath })` to create the review record
- [ ] UI shows upload progress (ActivityIndicator per image + overall)
- [ ] After submission, `ProfileScreen` shows "KYC en cours de vérification" badge
- [ ] `useKycStatus()` hook: polls `httpsCallable(functions, "getKycStatus")` every 60s until `verified` or `rejected`
- [ ] On `verified`: confetti animation + "Vous êtes vérifié" confirmation
- [ ] On `rejected`: shows rejection reason + "Réessayer" button

## Cloud Functions required
- `getKycUploadUrls({ docType, fileTypes[] })` → signed PUT URLs (valid 15 min)
- `submitKyc({ docType, frontPath, backPath })` → creates `kyc_submissions/{userId}` doc
- `getKycStatus()` → returns `{ status, rejectionReason? }`

## Implementation notes
- Use `expo-file-system` `readAsStringAsync` with `EncodingType.Base64` if blob is unavailable, OR convert URI to blob via `await (await fetch(uri)).blob()`
- Store `kycStatus` in AuthContext so ProfileScreen and HomeScreen both reflect it
