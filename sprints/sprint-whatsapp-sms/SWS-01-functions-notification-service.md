# SWS-01 — Notification Service CF + Provider Integrations

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SWS-01 |
| Branch | `feature/sws-01-notification-service` |
| Merges into | `dev` (mombongo-functions) |
| Estimate | 4h |
| Dependencies | Twilio account, Africa's Talking account |

## mombongo-functions

### Step 1 — Install dependencies

```bash
cd mombongo-functions
npm install twilio africastalking
```

### Step 2 — Set Firebase Function secrets

```bash
firebase functions:secrets:set TWILIO_ACCOUNT_SID
firebase functions:secrets:set TWILIO_AUTH_TOKEN
firebase functions:secrets:set TWILIO_WHATSAPP_FROM   # e.g. whatsapp:+14155238886
firebase functions:secrets:set AT_API_KEY
firebase functions:secrets:set AT_USERNAME
firebase functions:secrets:set AT_SENDER_ID           # optional
```

### Step 3 — Notification service module

Create `src/notifications/notificationService.ts`:

```typescript
import * as functions from 'firebase-functions'
import twilio from 'twilio'
import AfricasTalking from 'africastalking'

function twilioClient() {
  const sid   = functions.config().twilio?.account_sid   ?? process.env.TWILIO_ACCOUNT_SID
  const token = functions.config().twilio?.auth_token     ?? process.env.TWILIO_AUTH_TOKEN
  if (!sid || !token) throw new Error('Twilio credentials not set')
  return twilio(sid, token)
}

function atClient() {
  const key      = functions.config().at?.api_key  ?? process.env.AT_API_KEY  ?? ''
  const username = functions.config().at?.username ?? process.env.AT_USERNAME ?? 'sandbox'
  const AT = AfricasTalking({ apiKey: key, username })
  return AT.SMS
}

export interface NotifyPayload {
  phone: string          // E.164 format: +243xxxxxxxxx
  message: string        // Plain text (SMS + WhatsApp fallback body)
  templateSid?: string   // Twilio WhatsApp template SID (for approved templates)
  templateVars?: Record<string, string>
}

export async function sendWhatsApp(payload: NotifyPayload): Promise<void> {
  const from = functions.config().twilio?.whatsapp_from ?? process.env.TWILIO_WHATSAPP_FROM
  if (!from) throw new Error('TWILIO_WHATSAPP_FROM not set')
  const client = twilioClient()
  await client.messages.create({
    from,
    to:   `whatsapp:${payload.phone}`,
    body: payload.message,
  })
}

export async function sendSms(payload: NotifyPayload): Promise<void> {
  const sms = atClient()
  const senderId = functions.config().at?.sender_id ?? process.env.AT_SENDER_ID ?? undefined
  await sms.send({
    to:   [payload.phone],
    message: payload.message,
    from: senderId,
  })
}

// Attempts WhatsApp first; falls back to SMS on error.
export async function sendWithFallback(payload: NotifyPayload): Promise<void> {
  try {
    await sendWhatsApp(payload)
  } catch (err) {
    console.warn('WhatsApp send failed, falling back to SMS:', err)
    await sendSms(payload)
  }
}
```

### Step 4 — sendNotification onCall CF

Create `src/notifications/sendNotification.ts`:

```typescript
import * as functions from 'firebase-functions'
import * as admin from 'firebase-admin'
import { sendWhatsApp, sendSms, sendWithFallback } from './notificationService'

const db = admin.firestore()

export const sendNotification = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    // Only callable by admin-role users or internal (via other CFs using admin SDK)
    const uid = context.auth?.uid
    if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

    const userSnap = await db.collection('users').doc(uid).get()
    if (userSnap.data()?.role !== 'admin')
      throw new functions.https.HttpsError('permission-denied', 'Admin only')

    const {
      phone,
      message,
      channel = 'whatsapp_with_sms_fallback',
    } = data as {
      phone: string
      message: string
      channel?: 'whatsapp' | 'sms' | 'whatsapp_with_sms_fallback'
    }

    if (channel === 'whatsapp') await sendWhatsApp({ phone, message })
    else if (channel === 'sms') await sendSms({ phone, message })
    else await sendWithFallback({ phone, message })

    return { success: true }
  })
```

### Step 5 — Internal helper: notifyUser

Create `src/notifications/notifyUser.ts` — used internally by trigger CFs, not exposed via HTTP:

```typescript
import * as admin from 'firebase-admin'
import { sendWithFallback } from './notificationService'

const db = admin.firestore()

export async function notifyUser(uid: string, message: string): Promise<void> {
  const snap = await db.collection('users').doc(uid).get()
  const data = snap.data()

  const prefs = data?.notificationPrefs ?? {}
  const phone: string | null = data?.phone ?? null

  // Check opt-in (defaults to true if not set)
  const whatsappEnabled = prefs.whatsapp !== false
  const smsEnabled      = prefs.sms      !== false

  if (!phone) return  // no phone on file — FCM only

  if (whatsappEnabled || smsEnabled) {
    await sendWithFallback({ phone, message })
  }
}
```

Export `sendNotification` in `src/index.ts`.

---

## ✅ Definition of Done
- [ ] `twilio` and `africastalking` installed in `mombongo-functions`
- [ ] Firebase secrets set for Twilio + Africa's Talking
- [ ] `sendNotification` CF callable by admin role
- [ ] `notifyUser` helper usable internally by trigger CFs
- [ ] Sandbox test: send a WhatsApp message to your own number via Twilio sandbox
- [ ] Sandbox test: send an SMS to your own number via Africa's Talking sandbox
- [ ] `npm run build` exits 0

```bash
firebase deploy --only functions:sendNotification
```
