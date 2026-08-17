# SGN-04 — Password Reset, Email Verify & Profile Completion

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-04 |
| Repos | `mombongo-web`, `mombongo-mobile` |
| Branch | `feature/sgn-04-password-reset` |
| Merges into | `dev` |
| Priority | P0 — Launch blocker (users will forget their password; no recovery = churn) |
| Estimate | 3h |

## Why this matters
There is currently no "Forgot password" link on the login screen. A non-tech-savvy user who
forgets their password has no recovery path — they are permanently locked out. Firebase provides
`sendPasswordResetEmail` out of the box; this sprint just wires it up.
Profile photo upload is also missing, making the experience feel incomplete for all roles.

---

## Work items

### 1. "Mot de passe oublié" flow (web + mobile)

**On AuthScreen (login step)**:
- Add link below the password field: "Mot de passe oublié ?"
- Clicking it shows a mini form with just the email field
- Submit calls `sendPasswordResetEmail(auth, email)` from Firebase Auth SDK
- Success message: "Un lien de réinitialisation a été envoyé à votre adresse email."
- Error: if email not found → "Aucun compte trouvé avec cette adresse."

**Email template** (configured in Firebase Console → Authentication → Templates):
- Language: French
- Subject: "Mombongo — Réinitialisez votre mot de passe"
- Body: Clear instructions with a reset button

```tsx
// src/pages/AuthScreen.tsx — add to existing file
// Add a "forgotPassword" step to the existing step state machine
// step: "role" | "auth" | "forgot"
// Uses: import { sendPasswordResetEmail } from 'firebase/auth'
// auth is already imported from src/lib/firebase.ts (it IS exported from firebase.ts, unlike db)
```

### 2. Email verification banner
After signup, Firebase sends a verification email (if `sendEmailVerification` is called after
`createUserWithEmailAndPassword`). Currently this is not done.

Add to `createUserProfile` or the post-signup flow in AuthScreen:
```tsx
// After createUserWithEmailAndPassword:
if (userCredential.user && !userCredential.user.emailVerified) {
  await sendEmailVerification(userCredential.user)
}
```

Show a dismissible banner on Home screen if `user.emailVerified === false`:
- "Vérifiez votre adresse email. Nous avons envoyé un lien à [email]."
- Button: "Renvoyer l'email" → calls `sendEmailVerification` again

**Note**: Don't block app usage on email verification — just show a banner. Too strict = drop-off.

### 3. Profile photo upload
**Current state**: ProfileScreen shows a placeholder avatar. No way to upload a photo.

**Implementation**:
- Tap avatar → opens file picker (web) or image picker (mobile)
- Frontend calls `getProfilePhotoUploadUrl` CF → gets a signed Storage URL
- Frontend uploads file directly to signed URL (no Firebase Storage SDK in frontend)
- On upload success, call `updateProfile(auth.currentUser, { photoURL: downloadUrl })` via Firebase Auth SDK (allowed)
- Also update `users/{uid}.avatarUrl` via `updateUserProfile` CF call

**New CF needed**: `getProfilePhotoUploadUrl({ contentType })` — returns `{ uploadUrl, downloadUrl }`
```typescript
// mombongo-functions/src/profile/getProfilePhotoUploadUrl.ts
// Same pattern as getListingPhotoUploadUrl (already exists)
// Path: profile_photos/{uid}/{timestamp}.{ext}
// Returns signed PUT URL + the resulting public/signed GET URL
```

### 4. Profile completion score (optional but high impact for non-tech users)
On ProfileScreen, show a simple completion bar:
```
Profil: ████████░░ 80%
Complétez votre profil pour augmenter votre visibilité:
☐ Ajouter une photo
☑ Vérifier votre email
☑ Vérifier votre identité (KYC)
☐ Ajouter votre numéro mobile money
```

---

## Cloud Functions
- `getProfilePhotoUploadUrl({ contentType })` — signed Storage URL for profile photo
- `updateUserProfile({ avatarUrl?, phone?, mobileMoneyNumber? })` — updates user doc fields

---

## Acceptance criteria
- [ ] "Mot de passe oublié ?" link on login screen works — email sent, success message shown
- [ ] New users receive email verification email after signup
- [ ] Email verification banner appears on Home until verified
- [ ] Tapping avatar on ProfileScreen opens file picker → uploads → photo appears
- [ ] `npx tsc --noEmit` passes
- [ ] `npx vitest run` passes

---

## Implementation Status
NOT DONE
