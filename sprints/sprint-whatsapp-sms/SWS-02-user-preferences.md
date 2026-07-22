# SWS-02 — User Notification Preferences

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SWS-02 |
| Branch | `feature/sws-02-notification-prefs` |
| Merges into | `dev` (mombongo-web + mombongo-functions) |
| Estimate | 3h |
| Dependencies | SWS-01 (notifyUser helper exists) |

## Context

Users need to opt in/out of WhatsApp and SMS separately, and register a phone number if not already set on their profile. This is the prerequisite for any notification to land.

---

## mombongo-functions

### updateNotificationPrefs onCall

Create `src/notifications/updateNotificationPrefs.ts`:

```typescript
export const updateNotificationPrefs = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const {
      phone,
      whatsapp,
      sms,
      fcm,
    } = data as {
      phone?: string     // E.164
      whatsapp?: boolean
      sms?: boolean
      fcm?: boolean
    }

    const update: Record<string, unknown> = {
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    }

    if (phone !== undefined) update.phone = phone

    const prefs: Record<string, boolean> = {}
    if (whatsapp !== undefined) prefs.whatsapp = whatsapp
    if (sms       !== undefined) prefs.sms       = sms
    if (fcm       !== undefined) prefs.fcm        = fcm

    if (Object.keys(prefs).length > 0) {
      update['notificationPrefs'] = prefs
    }

    await db.collection('users').doc(uid).update(update)
    return { success: true }
  })
```

Export in `src/index.ts`.

---

## mombongo-web

### NotificationPrefsSection in ProfileScreen

Add a "Notifications" section to `src/pages/ProfileScreen.tsx`:

```tsx
// Inside the profile settings card, add after existing fields:

function NotificationPrefsSection() {
  const { t } = useTranslation()
  const qc = useQueryClient()
  const { data: profile } = useUserProfile()
  const [whatsapp, setWhatsapp] = useState(profile?.notificationPrefs?.whatsapp ?? true)
  const [sms, setSms] = useState(profile?.notificationPrefs?.sms ?? true)
  const [phone, setPhone] = useState(profile?.phone ?? '')
  const [saving, setSaving] = useState(false)

  const save = async () => {
    setSaving(true)
    try {
      const fn = httpsCallable(functions, 'updateNotificationPrefs')
      await fn({ phone: phone || undefined, whatsapp, sms })
      await qc.invalidateQueries({ queryKey: ['user-profile'] })
      toast.success(t('profile.prefsSaved'))
    } catch {
      toast.error(t('common.errorRetry'))
    } finally {
      setSaving(false)
    }
  }

  return (
    <div className="space-y-3">
      <h3 className="font-bold text-[14px] text-gray-800">{t('profile.notifications')}</h3>

      {/* Phone number field */}
      <div>
        <label className="block text-[12px] text-gray-500 mb-1">{t('profile.phone')}</label>
        <input
          value={phone}
          onChange={e => setPhone(e.target.value)}
          placeholder="+243 8xx xxx xxxx"
          className="w-full h-10 px-3 rounded-xl border border-gray-200 text-[13px]"
        />
        <p className="text-[11px] text-gray-400 mt-1">{t('profile.phoneHint')}</p>
      </div>

      {/* Toggle rows */}
      <ToggleRow
        label="WhatsApp"
        description={t('profile.whatsappDesc')}
        checked={whatsapp}
        onChange={setWhatsapp}
      />
      <ToggleRow
        label="SMS"
        description={t('profile.smsDesc')}
        checked={sms}
        onChange={setSms}
      />

      <button
        onClick={save}
        disabled={saving}
        className="h-10 px-5 bg-green-700 text-white rounded-xl text-[13px] font-bold disabled:opacity-50"
      >
        {saving ? t('common.saving') : t('common.save')}
      </button>
    </div>
  )
}

function ToggleRow({ label, description, checked, onChange }: {
  label: string; description: string; checked: boolean; onChange: (v: boolean) => void
}) {
  return (
    <div className="flex items-center justify-between py-2">
      <div>
        <p className="text-[13px] font-semibold text-gray-800">{label}</p>
        <p className="text-[11px] text-gray-400">{description}</p>
      </div>
      <button
        role="switch"
        aria-checked={checked}
        onClick={() => onChange(!checked)}
        className={`w-11 h-6 rounded-full transition ${checked ? 'bg-green-600' : 'bg-gray-200'}`}
      >
        <span className={`block w-5 h-5 bg-white rounded-full shadow transition translate-x-${checked ? '5' : '0.5'}`} />
      </button>
    </div>
  )
}
```

### i18n keys to add

```
profile.notifications     → "Notifications" (fr/en/ln)
profile.phone             → "Numéro de téléphone" / "Phone number"
profile.phoneHint         → "Requis pour WhatsApp et SMS. Format: +243 8xx xxx xxxx"
profile.whatsappDesc      → "Recevoir les alertes sur WhatsApp"
profile.smsDesc           → "Recevoir les alertes par SMS (hors connexion internet)"
profile.prefsSaved        → "Préférences enregistrées"
```

---

## ✅ Definition of Done
- [ ] `updateNotificationPrefs` CF updates `users.phone` and `users.notificationPrefs`
- [ ] ProfileScreen shows phone field + WhatsApp/SMS toggles
- [ ] Saving prefs in dev mode (isDevMode) shows toast without CF call
- [ ] i18n keys added in fr/en/ln
- [ ] `npx vitest run` passes
- [ ] `npm run build` exits 0
