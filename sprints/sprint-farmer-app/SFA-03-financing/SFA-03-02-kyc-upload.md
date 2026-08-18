# SFA-03-02 — KYC Document Capture (Farmer App)

## Context
Port of SM-04. Farmers must upload KYC documents (national ID, land title, bank statement) before a financing application can be approved. The upload flow: capture photo or pick from gallery → upload via a CF that returns a signed GCS URL → the app PUT's the image directly to GCS → the CF marks the document as uploaded.

**No direct Firebase Storage SDK calls from the app.** The upload goes:
1. App calls `getKycUploadUrl` CF → receives a signed PUT URL + document ID
2. App HTTP PUT's the image to the signed URL
3. App calls `confirmKycUpload` CF with document ID → CF validates + updates Firestore status

## Scope
- Create `src/screens/kyc/KycUploadScreen.tsx`
- Create `src/hooks/useKycUpload.ts` — orchestrates the 3-step upload
- Show upload status per document type (national_id, land_title, bank_statement)
- Accessible from Profile screen + from financing apply flow (step gating)

## Cloud Functions required
- `getKycUploadUrl` — input: `{ docType: string }` → output: `{ uploadUrl: string; documentId: string }`
- `confirmKycUpload` — input: `{ documentId: string }` → output: `{ status: string }`
- `getKycStatus` — input: void → output: `{ documents: KycDocument[] }`

## Files to create
- `src/hooks/useKycUpload.ts`
- `src/screens/kyc/KycUploadScreen.tsx`

## Implementation

### `src/hooks/useKycUpload.ts`
```typescript
import * as ImagePicker from 'expo-image-picker'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'

type DocType = 'national_id' | 'land_title' | 'bank_statement'

export function useKycUpload() {
  const [uploading, setUploading] = useState<DocType | null>(null)

  const uploadDocument = async (docType: DocType) => {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      quality: 0.8,
      base64: false,
    })
    if (result.canceled) return

    setUploading(docType)
    try {
      // Step 1: get signed URL
      const { data } = await httpsCallable<{ docType: string }, { uploadUrl: string; documentId: string }>(
        functions, 'getKycUploadUrl'
      )({ docType })

      // Step 2: PUT image directly to GCS
      const asset = result.assets[0]
      const response = await fetch(asset.uri)
      const blob = await response.blob()
      await fetch(data.uploadUrl, {
        method: 'PUT',
        body: blob,
        headers: { 'Content-Type': asset.mimeType ?? 'image/jpeg' },
      })

      // Step 3: confirm upload
      await httpsCallable(functions, 'confirmKycUpload')({ documentId: data.documentId })
    } finally {
      setUploading(null)
    }
  }

  return { uploadDocument, uploading }
}
```

### KycUploadScreen.tsx
```typescript
const DOC_TYPES: Array<{ key: DocType; label: string; description: string }> = [
  { key: 'national_id', label: 'Carte nationale d\'identité', description: 'Photo recto-verso' },
  { key: 'land_title', label: 'Titre foncier', description: 'Document officiel' },
  { key: 'bank_statement', label: 'Relevé bancaire', description: '3 derniers mois' },
]

// Each doc type: status chip (pending/uploaded/verified) + upload button
// Camera capture option via ImagePicker.launchCameraAsync
// Upload progress indicator when uploading
```

## Install command
```bash
npx expo install expo-image-picker
```

## Acceptance criteria
- [ ] KYC screen accessible from Profile
- [ ] Each document type shows its current status (pending/uploaded/verified)
- [ ] Tapping upload button opens image picker (gallery or camera)
- [ ] Image PUT's to signed URL without error
- [ ] After confirm, status updates to "uploaded"
- [ ] Admin can see uploaded documents in admin backoffice

## Smoke test
1. Open Profile → KYC
2. Tap "Télécharger" on national_id → select image from gallery
3. Confirm progress indicator appears then disappears
4. Confirm status chip updates to "Téléchargé"
5. Firebase console → verify KYC document record exists in Firestore
