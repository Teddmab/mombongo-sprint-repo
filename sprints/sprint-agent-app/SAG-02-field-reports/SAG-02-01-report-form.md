# SAG-02-01 — Field Report Form + Photo Upload (Agent App)

## Context
Port of web sprint SG-05. Mobile `AgentReportScreen.tsx` exists but calls `useAgentReports` from local mock. This sprint wires it to real CFs and adds photo capture (a key differentiator — web can't take photos, mobile can).

The `submitAgentReport` CF (deployed in SG-05) accepts:
- `farmerId`, `exploitationId`
- `visitDate`, `notes`, `recommendations`
- `cropHealthScore` (1–5)
- `photoUrls` (array of signed URL strings already uploaded to GCS)

Photo upload: same signed URL pattern as KYC (SFA-03-02) — call `getReportPhotoUploadUrl` CF → PUT to signed URL → include URL in report payload.

## Scope
- Create `src/screens/reports/AgentReportFormScreen.tsx` (multi-step form)
- Create `src/hooks/useSubmitReport.ts`
- Add photo capture step (camera OR gallery, max 5 photos)
- Pre-fill farmer selector when navigating from farmer detail or visit plan
- Numeric crop health score (1–5 stars)

## Cloud Functions required
- `submitAgentReport` — already deployed (SG-05)
- `getReportPhotoUploadUrl` — new; input: `{ count: number }` → output: `{ uploadUrls: Array<{ uploadUrl: string; fileId: string }> }`

## Files to create
- `src/hooks/useSubmitReport.ts`
- `src/screens/reports/AgentReportFormScreen.tsx`

## Implementation

### `src/hooks/useSubmitReport.ts`
```typescript
import * as ImagePicker from 'expo-image-picker'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'

export function useSubmitReport() {
  const qc = useQueryClient()
  const [uploadProgress, setUploadProgress] = useState(0)

  const uploadPhotos = async (assets: ImagePicker.ImagePickerAsset[]): Promise<string[]> => {
    if (!assets.length) return []
    const { data } = await httpsCallable<{ count: number }, { uploadUrls: Array<{ uploadUrl: string; fileId: string }> }>(
      functions, 'getReportPhotoUploadUrl'
    )({ count: assets.length })

    const photoUrls: string[] = []
    for (let i = 0; i < assets.length; i++) {
      const asset = assets[i]
      const { uploadUrl, fileId } = data.uploadUrls[i]
      const blob = await (await fetch(asset.uri)).blob()
      await fetch(uploadUrl, { method: 'PUT', body: blob, headers: { 'Content-Type': 'image/jpeg' } })
      photoUrls.push(fileId)
      setUploadProgress((i + 1) / assets.length)
    }
    return photoUrls
  }

  const submit = useMutation({
    mutationFn: async (payload: ReportPayload & { photos: ImagePicker.ImagePickerAsset[] }) => {
      const { photos, ...rest } = payload
      const photoUrls = await uploadPhotos(photos)
      return httpsCallable(functions, 'submitAgentReport')({ ...rest, photoUrls })
    },
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['agentReports'] })
      qc.invalidateQueries({ queryKey: ['agentFarmers'] })
    },
  })

  return { submit, uploadProgress }
}
```

### AgentReportFormScreen.tsx
```typescript
// Step 1: Farmer selector (pre-filled if navigated from farmer detail)
// Step 2: Visit details — date, crop health stars (1–5), notes textarea
// Step 3: Photos — camera + gallery picker, max 5 photos, thumbnail preview grid
// Step 4: Recommendations — free text
// Step 5: Review + submit

// Progress bar at top showing current step
// "Enregistrer brouillon" option (local AsyncStorage save)
```

## Install command
```bash
npx expo install expo-image-picker expo-camera
```

## Acceptance criteria
- [ ] Report form accessible from AgentHomeScreen, farmer detail, and Rapports tab
- [ ] Farmer pre-filled when navigating from farmer detail
- [ ] Photo capture works (camera + gallery), max 5 photos, thumbnails shown
- [ ] Photos upload via signed URL before report submitted
- [ ] Submit calls `submitAgentReport` CF with photo URLs
- [ ] After submit, farmer's "last visit date" updates in farmer list

## Smoke test
1. Navigate to report form from farmer detail
2. Confirm farmer is pre-filled in step 1
3. Take 2 photos via camera
4. Complete all steps → submit
5. Firebase console → verify report document with photoUrls array
6. Return to farmer list → confirm "last visit" shows today
