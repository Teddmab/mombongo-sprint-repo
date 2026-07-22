# SWS-04 — Admin Broadcast Panel

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SWS-04 |
| Branch | `feature/sws-04-admin-broadcast` |
| Merges into | `dev` (mombongo-functions + mombongo-admin) |
| Estimate | 2–4h |
| Dependencies | SWS-01 (sendWhatsApp + sendSms), SWS-02 (users have phone + prefs) |

## Context

Ops team needs to send targeted notifications to user segments — e.g. "all farmers in Bandundu" or "all merchants with open buyer orders". This is an admin-only feature.

---

## mombongo-functions

### broadcastNotification onCall

Create `src/notifications/broadcastNotification.ts`:

```typescript
export const broadcastNotification = functions
  .region('europe-west1')
  .runWith({ timeoutSeconds: 540, memory: '512MB' })
  .https.onCall(async (data, context) => {
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const callerSnap = await db.collection('users').doc(uid).get()
    if (callerSnap.data()?.role !== 'admin')
      throw new functions.https.HttpsError('permission-denied', 'Admin only')

    const {
      message,
      channel = 'whatsapp_with_sms_fallback',
      segment,   // 'all' | 'farmers' | 'investors' | 'merchants' | 'agents'
      province,  // optional filter
    } = data as {
      message: string
      channel: 'whatsapp' | 'sms' | 'whatsapp_with_sms_fallback'
      segment: string
      province?: string
    }

    let q = db.collection('users').where('isActive', '==', true) as FirebaseFirestore.Query
    if (segment !== 'all') q = q.where('role', '==', segment.replace('s', ''))  // 'farmers' → 'farmer'
    if (province) q = q.where('province', '==', province)

    const snap = await q.get()
    const results = { sent: 0, failed: 0, skipped: 0 }

    for (const doc of snap.docs) {
      const user = doc.data()
      const phone: string | null = user.phone ?? null
      const prefs = user.notificationPrefs ?? {}

      if (!phone) { results.skipped++; continue }

      const whatsappOk = prefs.whatsapp !== false
      const smsOk      = prefs.sms      !== false

      if (!whatsappOk && !smsOk) { results.skipped++; continue }

      try {
        if (channel === 'whatsapp' && whatsappOk) {
          await sendWhatsApp({ phone, message })
        } else if (channel === 'sms' && smsOk) {
          await sendSms({ phone, message })
        } else {
          await sendWithFallback({ phone, message })
        }
        results.sent++
      } catch {
        results.failed++
      }
    }

    // Log the broadcast
    await db.collection('admin_broadcasts').add({
      message, channel, segment, province: province ?? null,
      sentCount: results.sent, failedCount: results.failed, skippedCount: results.skipped,
      sentBy: uid,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    })

    return results
  })
```

Export in `src/index.ts`.

---

## mombongo-admin

### AdminBroadcast panel (new tab in AdminAlerts or standalone page)

Add a **Diffusion** tab to `src/pages/Admin.tsx` (AdminAlerts section) or create `src/pages/AdminBroadcast.tsx`:

```tsx
export function AdminBroadcast() {
  const [message, setMessage] = useState('')
  const [channel, setChannel] = useState<'whatsapp_with_sms_fallback' | 'whatsapp' | 'sms'>('whatsapp_with_sms_fallback')
  const [segment, setSegment] = useState('all')
  const [province, setProvince] = useState('')
  const [result, setResult] = useState<{ sent: number; failed: number; skipped: number } | null>(null)
  const [loading, setLoading] = useState(false)

  const send = async () => {
    if (!message.trim()) return
    if (!window.confirm(`Envoyer à "${segment}" via ${channel} ?\n\n"${message}"`)) return
    setLoading(true)
    try {
      const fn = httpsCallable(functions, 'broadcastNotification')
      const res = await fn({ message, channel, segment, province: province || undefined })
      setResult(res.data as any)
    } catch (e: any) {
      alert(e.message)
    } finally {
      setLoading(false)
    }
  }

  // Render: segment select, province input, channel select, message textarea, send button
  // Result card: "Envoyé: X · Échec: Y · Ignoré: Z"
}
```

Wire to `/admin/broadcast` route and add to NAV as **Diffusion** with the `Send` (Lucide) icon.

---

## ✅ Definition of Done
- [ ] `broadcastNotification` CF filters users by segment + province, sends, logs to `admin_broadcasts`
- [ ] Admin UI has message composer + segment/channel selectors + confirmation dialog
- [ ] Result card shows sent/failed/skipped counts
- [ ] Broadcast history visible (last 10 entries from `admin_broadcasts`)
- [ ] `npm run build` exits 0 (functions + admin)

```bash
firebase deploy --only functions:broadcastNotification
```
