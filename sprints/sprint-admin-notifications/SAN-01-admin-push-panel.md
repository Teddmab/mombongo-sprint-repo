# SAN-01 — Admin Notification Panel (FCM Push + Manual Price Push Trigger)

## Context
`sendMorningPricePush` runs on a cron at 06:30 WAT. There is currently no way for the admin to:
- Test that FCM push reaches a specific user (for debugging FCM token issues)
- Trigger the morning price push on demand (e.g. after seeding new province_prices data)
- Send a one-off FCM push to a specific user or role segment

The existing `broadcastNotification` CF (SWS-04) handles WhatsApp/SMS only. This story adds FCM push capabilities to the admin toolbox.

## Scope

### mombongo-functions
Two new admin-only CFs:

**`adminSendPush`** — send a custom FCM push to a single UID or an entire role segment.  
**`adminTriggerMorningPricePush`** — invokes the same logic as `sendMorningPricePush` but as an onCall CF so the admin can run it at any time.

### mombongo-admin
New **Notifications** tab in the admin panel with two sections:
1. **Test push** — send a custom FCM push to a single user by UID or email
2. **Trigger price push** — run the morning price push for all eligible farmers right now

---

## mombongo-functions

### `src/notifications/adminSendPush.ts`
```typescript
import { functions, admin } from '../lib/admin'
import { sendPush } from './sendPush'

const db = admin.firestore()

export const adminSendPush = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    if (!context.auth?.uid)
      throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const callerSnap = await db.collection('users').doc(context.auth.uid).get()
    if (callerSnap.data()?.role !== 'admin')
      throw new functions.https.HttpsError('permission-denied', 'Admin only')

    const { targetUid, targetRole, title, body, screen, crop, province } = data as {
      targetUid?: string
      targetRole?: string  // 'farmer' | 'agent' | 'investor' | 'merchant'
      title: string
      body: string
      screen?: string
      crop?: string
      province?: string
    }

    if (!title || !body)
      throw new functions.https.HttpsError('invalid-argument', 'title and body required')

    const pushData: Record<string, string> = {}
    if (screen)   pushData.screen   = screen
    if (crop)     pushData.crop     = crop
    if (province) pushData.province = province

    const results = { sent: 0, failed: 0 }

    if (targetUid) {
      // Single user
      try {
        await sendPush(targetUid, title, body, pushData)
        results.sent++
      } catch {
        results.failed++
      }
    } else if (targetRole) {
      // All users with this role
      const snap = await db.collection('users').where('role', '==', targetRole).get()
      await Promise.all(snap.docs.map(async doc => {
        try {
          await sendPush(doc.id, title, body, pushData)
          results.sent++
        } catch {
          results.failed++
        }
      }))
    } else {
      throw new functions.https.HttpsError('invalid-argument', 'targetUid or targetRole required')
    }

    // Log to admin_push_log
    await db.collection('admin_push_log').add({
      title, body, screen: screen ?? null, targetUid: targetUid ?? null, targetRole: targetRole ?? null,
      results,
      sentBy: context.auth.uid,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    })

    return results
  })
```

### `src/notifications/adminTriggerMorningPricePush.ts`
```typescript
import { functions, admin } from '../lib/admin'

const db = admin.firestore()

// Re-export the core morning price logic as a callable so admin can trigger on demand.
// This imports the same underlying function logic from sendMorningPricePush.ts.
// To avoid code duplication, extract the core logic into a shared function:
//   sendMorningPricePushCore() → called by both the scheduled CF and this onCall CF.

import { sendMorningPricePushCore } from './sendMorningPricePush'

export const adminTriggerMorningPricePush = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    if (!context.auth?.uid)
      throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const callerSnap = await db.collection('users').doc(context.auth.uid).get()
    if (callerSnap.data()?.role !== 'admin')
      throw new functions.https.HttpsError('permission-denied', 'Admin only')

    functions.logger.info(`adminTriggerMorningPricePush: triggered manually by ${context.auth.uid}`)

    await sendMorningPricePushCore()

    return { success: true, triggeredAt: new Date().toISOString() }
  })
```

### Required refactor in `sendMorningPricePush.ts`
Extract the handler body into an exported `sendMorningPricePushCore()` function so both the scheduled CF and the onCall admin CF can call it:

```typescript
// Export the core logic
export async function sendMorningPricePushCore(): Promise<void> {
  // ... existing handler body (all the Firestore queries + sendPush calls) ...
}

// Keep the scheduled CF as a thin wrapper
export const sendMorningPricePush = functions
  .region('europe-west1')
  .pubsub.schedule('30 6 * * *')
  .timeZone('Africa/Kinshasa')
  .onRun(() => sendMorningPricePushCore())
```

### Export in `src/index.ts`
```typescript
// ─── Admin: Notification tools ────────────────────────────────────────────────
export { adminSendPush }                from './notifications/adminSendPush'
export { adminTriggerMorningPricePush } from './notifications/adminTriggerMorningPricePush'
```

---

## mombongo-admin

### New page: `src/pages/AdminNotifications.tsx`

```tsx
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'

const adminSendPushFn                = httpsCallable(functions, 'adminSendPush')
const adminTriggerMorningPricePushFn = httpsCallable(functions, 'adminTriggerMorningPricePush')

export function AdminNotifications() {
  return (
    <div className="space-y-8 p-6">
      <h1 className="text-2xl font-semibold">Notifications</h1>

      {/* Section 1: Test push */}
      <TestPushSection />

      {/* Section 2: Trigger morning price push */}
      <MorningPricePushSection />

      {/* Section 3: Push log */}
      <PushLogSection />
    </div>
  )
}

function TestPushSection() {
  const [mode, setMode]     = useState<'uid' | 'role'>('uid')
  const [target, setTarget] = useState('')
  const [title, setTitle]   = useState('')
  const [body, setBody]     = useState('')
  const [screen, setScreen] = useState('')
  const [result, setResult] = useState<{ sent: number; failed: number } | null>(null)
  const [loading, setLoading] = useState(false)

  const SCREENS = ['market', 'financement', 'bourse', 'exploitation', 'academia']
  const ROLES   = ['farmer', 'agent', 'investor', 'merchant']

  const send = async () => {
    if (!title || !body || !target) return
    const confirmed = window.confirm(`Envoyer "${title}" → ${mode === 'uid' ? target : `tous les ${target}s`} ?`)
    if (!confirmed) return

    setLoading(true)
    try {
      const payload: Record<string, string> = { title, body }
      if (screen) payload.screen = screen
      if (mode === 'uid')  payload.targetUid  = target
      if (mode === 'role') payload.targetRole = target

      const res = await adminSendPushFn(payload)
      setResult(res.data as { sent: number; failed: number })
    } catch (e: any) {
      alert(e.message)
    } finally {
      setLoading(false)
    }
  }

  return (
    <section className="border rounded-lg p-4 space-y-4">
      <h2 className="font-medium">Envoyer un push test</h2>

      {/* Mode toggle: UID vs Role */}
      <div className="flex gap-2">
        <button className={mode === 'uid' ? 'btn-primary' : 'btn-ghost'} onClick={() => setMode('uid')}>Par UID</button>
        <button className={mode === 'role' ? 'btn-primary' : 'btn-ghost'} onClick={() => setMode('role')}>Par rôle</button>
      </div>

      {mode === 'uid'
        ? <input placeholder="UID utilisateur" value={target} onChange={e => setTarget(e.target.value)} className="input w-full" />
        : <select value={target} onChange={e => setTarget(e.target.value)} className="input w-full">
            <option value="">Choisir un rôle…</option>
            {ROLES.map(r => <option key={r} value={r}>{r}</option>)}
          </select>
      }

      <input placeholder="Titre" value={title} onChange={e => setTitle(e.target.value)} className="input w-full" />
      <textarea placeholder="Corps du message" value={body} onChange={e => setBody(e.target.value)} className="input w-full" rows={2} />

      <select value={screen} onChange={e => setScreen(e.target.value)} className="input w-full">
        <option value="">Écran cible (optionnel)</option>
        {SCREENS.map(s => <option key={s} value={s}>{s}</option>)}
      </select>

      <button onClick={send} disabled={loading || !title || !body || !target} className="btn-primary w-full">
        {loading ? 'Envoi…' : 'Envoyer'}
      </button>

      {result && (
        <p className="text-sm text-green-600">✓ Envoyé: {result.sent} · Échec: {result.failed}</p>
      )}
    </section>
  )
}

function MorningPricePushSection() {
  const [loading, setLoading] = useState(false)
  const [result, setResult]   = useState<string | null>(null)

  const trigger = async () => {
    const confirmed = window.confirm('Déclencher le push prix du matin pour tous les agriculteurs éligibles ?')
    if (!confirmed) return
    setLoading(true)
    try {
      const res = await adminTriggerMorningPricePushFn({})
      setResult((res.data as any).triggeredAt)
    } catch (e: any) {
      alert(e.message)
    } finally {
      setLoading(false)
    }
  }

  return (
    <section className="border rounded-lg p-4 space-y-4">
      <h2 className="font-medium">Prix du matin</h2>
      <p className="text-sm text-muted-foreground">
        Déclenche manuellement le push prix du marché (normalement envoyé à 06h30 WAT).
        Utile pour tester après avoir mis à jour les prix dans <code>province_prices</code>.
      </p>
      <button onClick={trigger} disabled={loading} className="btn-primary">
        {loading ? 'En cours…' : '▶ Déclencher maintenant'}
      </button>
      {result && <p className="text-sm text-green-600">✓ Déclenché à {new Date(result).toLocaleString('fr-FR')}</p>}
    </section>
  )
}
```

### Route + nav
Add to `src/App.tsx` (admin):
```tsx
<Route path="/admin/notifications" element={<AdminNotifications />} />
```
Add to nav sidebar:
```tsx
{ label: 'Notifications', path: '/admin/notifications', icon: Bell }
```

---

## Acceptance criteria
- [ ] `adminSendPush` CF: admin can push to a single UID → notification arrives on the user's device
- [ ] `adminSendPush` CF: admin can push to all `farmer` accounts → all registered devices receive it
- [ ] `adminTriggerMorningPricePush` CF: triggers the full morning push logic and returns success
- [ ] Both CFs reject non-admin callers with `permission-denied`
- [ ] Admin UI has Test push section + Morning price push trigger button
- [ ] Push log (`admin_push_log`) records each manual send with sentBy, results, timestamp
- [ ] After triggering morning push from admin UI: farmers with FCM tokens receive a push within 30s

## Smoke test — SU-01-01 end-to-end via admin panel

**Prerequisites (⚠️ manual — confirm with Teddy before testing):**
1. Deploy: `firebase deploy --only functions:sendMorningPricePush,functions:adminSendPush,functions:adminTriggerMorningPricePush`
2. Firebase console → Firestore → `config/exchange_rate` → set `usdToCdf: 2800`
3. Firebase console → Firestore → `province_prices` → create doc: `{ commodity: "Maïs", province: "Kinshasa", pricePerKgCdf: 450, updatedAt: <now> }`
4. Sign in to mombongo-web as a farmer → push permission granted → verify `users/{uid}.fcmToken` is set

**Test steps:**
1. Open mombongo-admin → Notifications tab
2. Paste farmer UID in Test push → title "Test" → body "Bonjour" → Send
3. Confirm farmer's browser/device receives the push ✓
4. Set screen to "market" → Send again → confirm tapping push navigates to Market screen ✓
5. Click "Déclencher maintenant" (morning price push) → confirm success ✓
6. Check farmer browser → confirm price push notification arrives ("Maïs à 450 FC/kg en Kinshasa") ✓
7. Tap notification → confirm navigates to Market screen with `?crop=Maïs&province=Kinshasa` ✓

## Deploy command
```bash
firebase deploy --only \
  functions:sendMorningPricePush \
  functions:adminSendPush \
  functions:adminTriggerMorningPricePush
```
