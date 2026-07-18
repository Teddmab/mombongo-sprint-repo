# SG-04 — KYC: User-Facing Document Upload & Status Tracking

## Why this matters
KYC is managed entirely in the admin panel. Users have no way to upload documents or know what's happening with their verification. For a financial app in DRC, KYC is a regulatory requirement — users need to be guided through it clearly and simply.

## Design principle
Make it feel like filling in a form at a bank, but on a phone. One document at a time. Clear progress. Plain language.

## Current state
- No KYC screen or flow in `mombongo-web` at all
- `kycStatus` field exists on `users/{uid}` doc (none / pending / approved / rejected)
- `kycRejectionReason` field exists but is never shown to the user
- Admin validates/rejects via AdminKyc (currently mock, see SA-05)

## Work items

### 1. KYC entry point
- Banner on Home screen for users where `kycStatus !== 'approved'`:
  - Grey banner: "Vérifiez votre identité pour investir et retirer des fonds"
  - Tap → navigates to `/kyc`
- After KYC is approved, banner disappears permanently

### 2. KYC screen (`/kyc`)
Three visual states:

**State A — Not started / rejected**
- Explanation in plain language (2–3 sentences max): why KYC is needed, what it unlocks
- Document checklist: which documents are accepted (Carte nationale d'identité, Passeport, Permis de conduire)
- "Commencer" button → goes to upload flow
- If rejected: show rejection reason in a red card, then "Recommencer" button

**State B — Pending review**
- Big yellow clock icon
- "Vos documents sont en cours de vérification. Délai habituel : 24–48 heures."
- No action needed — just wait

**State C — Approved**
- Big green checkmark
- "Votre identité est vérifiée ✓"
- This screen is accessible from profile but the banner on Home is gone

### 3. Document upload flow (3 steps)
**Step 1 — Choisir le document**
- 3 large buttons: Carte d'identité / Passeport / Permis de conduire

**Step 2 — Prendre une photo**
- Front of document: camera button (opens device camera or file picker)
- Back of document (if applicable): second camera button
- Preview thumbnails — user can retake before submitting

**Step 3 — Soumettre**
- Summary: document type + thumbnails
- "Envoyer pour vérification" button
- Calls `submitKycDocuments` CF

### 4. Cloud Functions needed
- `getKycUploadUrls(uid, fileCount)` — returns N signed Storage URLs for upload
- Frontend uploads files directly to Storage using the signed URLs
- `submitKycDocuments(uid, { documentType, photoUrls[] })` — creates/updates `kyc_submissions/{uid}` doc with `status: 'pending'`; sets `users/{uid}.kycStatus = 'pending'`

### 5. Push notification on KYC result
- When admin approves/rejects: send push notification to user (via FCM, see S6-02)
- Approval: "🎉 Votre identité a été vérifiée ! Vous pouvez maintenant investir."
- Rejection: "⚠️ Vérification refusée. Ouvrez l'app pour voir la raison."

## Acceptance criteria
- [ ] KYC banner appears on Home for unverified users, disappears once approved
- [ ] User can upload 1–2 photos of their ID document
- [ ] Upload uses signed Storage URL (no direct Firebase Storage SDK)
- [ ] After upload, status shows "En cours de vérification"
- [ ] Rejection reason is shown clearly if rejected
- [ ] Push notification sent on approval and rejection
