# S6-02 — Notifications — FCM Push Notifications

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S6-02 |
| Sprint | Sprint 6 — Payments & Notifications |
| Branch | `feature/s6-02-fcm-notifications` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S6-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | 🔨 Active | onInvestmentCreated Firestore trigger + onHarvestDue scheduled function |
| `mombongo-web` | 🔨 Active | FCM token registration, NotificationBell component |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-functions

### Step 1 — Send notification helper

Create `src/notifications/sendNotification.ts`:

```typescript
import * as admin from 'firebase-admin'

export async function sendPushToUser(
  userId: string,
  payload: { title: string; body: string; data?: Record<string, string> }
) {
  const userSnap = await admin.firestore().collection('users').doc(userId).get()
  const fcmToken: string | undefined = userSnap.data()?.fcmToken

  if (!fcmToken) return   // user hasn't granted notification permission

  await admin.messaging().send({
    token: fcmToken,
    notification: { title: payload.title, body: payload.body },
    data: payload.data ?? {},
    webpush: {
      notification: {
        icon:  '/icons/icon-192x192.png',
        badge: '/icons/badge-72x72.png',
        click_action: 'https://app.mombongo.com',
      },
    },
  })
}
```

### Step 2 — onInvestmentCreated Firestore trigger

```typescript
// src/notifications/investmentTriggers.ts
export const onInvestmentCreated = functions.firestore
  .document('investments/{investmentId}')
  .onCreate(async snap => {
    const inv = snap.data()
    await sendPushToUser(inv.investorId, {
      title: '✅ Investissement confirmé',
      body:  `Votre investissement de ${formatUsd(inv.amountUsd)} dans ${inv.productName} est actif.`,
      data:  { type: 'investment_created', investmentId: snap.id },
    })
  })

export const onBourseInvestmentCreated = functions.firestore
  .document('bourse_investments/{investmentId}')
  .onCreate(async snap => {
    const inv = snap.data()
    await sendPushToUser(inv.investorId, {
      title: '✅ Investissement bourse confirmé',
      body:  `Votre investissement de ${formatCdf(inv.amountCdf)} dans ${inv.route} est actif.`,
      data:  { type: 'bourse_investment_created', investmentId: snap.id },
    })
  })
```

### Step 3 — onHarvestDue scheduled function

```typescript
// src/notifications/scheduledNotifications.ts
export const onHarvestDue = functions.pubsub
  .schedule('every 24 hours')
  .onRun(async () => {
    const tomorrow = new Date()
    tomorrow.setDate(tomorrow.getDate() + 1)
    const dayAfter = new Date(tomorrow)
    dayAfter.setDate(dayAfter.getDate() + 1)

    // Find investments with harvest date tomorrow
    const snap = await db.collection('investments')
      .where('harvestDate', '>=', admin.firestore.Timestamp.fromDate(tomorrow))
      .where('harvestDate', '<',  admin.firestore.Timestamp.fromDate(dayAfter))
      .where('status', '==', 'active')
      .get()

    const notifications = snap.docs.map(doc => {
      const inv = doc.data()
      return sendPushToUser(inv.investorId, {
        title: '🌾 Récolte demain !',
        body:  `Votre investissement dans ${inv.productName} arrive à maturité demain.`,
        data:  { type: 'harvest_due', investmentId: doc.id },
      })
    })

    await Promise.allSettled(notifications)
  })
```

Export all in `src/index.ts`.

---

## mombongo-web

### Step 1 — FCM token registration

Add to `src/lib/firebase.ts`:

```typescript
import { getMessaging, getToken, onMessage } from 'firebase/messaging'

export const messaging = getMessaging(app)
export const FCM_VAPID_KEY = import.meta.env.VITE_FCM_VAPID_KEY

export async function requestNotificationPermission(userId: string): Promise<string | null> {
  if (!('Notification' in window)) return null
  const permission = await Notification.requestPermission()
  if (permission !== 'granted') return null

  const token = await getToken(messaging, { vapidKey: FCM_VAPID_KEY })

  // Save FCM token to Firestore user doc
  const { doc, updateDoc } = await import('firebase/firestore')
  await updateDoc(doc(db, 'users', userId), { fcmToken: token })

  return token
}
```

Add `VITE_FCM_VAPID_KEY=your_web_push_certificate_key` to `.env.local`.

Add `public/firebase-messaging-sw.js` (service worker for background notifications):
```javascript
importScripts('https://www.gstatic.com/firebasejs/10.0.0/firebase-app-compat.js')
importScripts('https://www.gstatic.com/firebasejs/10.0.0/firebase-messaging-compat.js')

firebase.initializeApp({
  apiKey:            self.FIREBASE_CONFIG_API_KEY,
  authDomain:        self.FIREBASE_CONFIG_AUTH_DOMAIN,
  projectId:         self.FIREBASE_CONFIG_PROJECT_ID,
  storageBucket:     self.FIREBASE_CONFIG_STORAGE_BUCKET,
  messagingSenderId: self.FIREBASE_CONFIG_MESSAGING_SENDER_ID,
  appId:             self.FIREBASE_CONFIG_APP_ID,
})

const messaging = firebase.messaging()
messaging.onBackgroundMessage(payload => {
  self.registration.showNotification(payload.notification.title, {
    body: payload.notification.body,
    icon: '/icons/icon-192x192.png',
  })
})
```

### Step 2 — NotificationBell component

`src/components/NotificationBell.tsx` — request permission on first click, show unread badge.

```tsx
export function NotificationBell() {
  const { user } = useAuth()
  const [permitted, setPermitted] = useState(Notification.permission === 'granted')

  async function handleClick() {
    if (!user?.uid) return
    if (!permitted) {
      const token = await requestNotificationPermission(user.uid)
      if (token) setPermitted(true)
    }
  }

  return (
    <button
      data-testid="notification-bell"
      onClick={handleClick}
      className="relative p-2"
      aria-label={t('notifications.enable')}
    >
      <span className="text-xl">{permitted ? '🔔' : '🔕'}</span>
    </button>
  )
}
```

Add `NotificationBell` to `DesktopShell` header and `BottomNav` `TopNav` area.

### Step 3 — i18n keys

```
notifications.enable   → "Activer les notifications" / "Enable notifications"
notifications.enabled  → "Notifications activées" / "Notifications enabled"
notifications.denied   → "Notifications refusées — activez dans les paramètres du navigateur" / "Notifications denied — enable in browser settings"
```

---

## ✅ Definition of Done
- [ ] `onInvestmentCreated` trigger sends push on new `investments` doc
- [ ] `onHarvestDue` runs daily and notifies investors of upcoming harvests
- [ ] Web app requests notification permission and saves FCM token to user doc
- [ ] Background notifications work via `firebase-messaging-sw.js`
- [ ] `data-testid="notification-bell"` present in shell
- [ ] `npm run build` exits 0

```bash
firebase deploy --only functions:onInvestmentCreated,onBourseInvestmentCreated,onHarvestDue
git commit -m "feat(s6-02): FCM push notifications — investment + harvest triggers"
git push origin feature/s6-02-fcm-notifications
```
