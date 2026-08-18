# SFA-00-04 — Expo Notifications + FCM Token Registration (Farmer App)

## Context
Port of SM-05-00. The web PWA uses `firebase/messaging` + service worker for push. Mobile uses `expo-notifications` which wraps FCM on Android and APNs on iOS.

The web `sendMorningPricePush` CF (already deployed in SU-01-01) reads `users/{uid}.fcmToken` (or `fcmTokens[]`). On mobile we register the Expo push token via the same `registerFcmToken` CF — this ensures the existing CF works for both web and mobile without changes.

**Note on Expo Push vs FCM directly:** Expo uses its own push proxy to route to FCM/APNs. However, our `sendPush` utility in `mombongo-functions` calls the FCM API directly. For native apps we need the raw FCM token, not the Expo push token. Use `getDevicePushTokenAsync()` (raw FCM token) rather than `getExpoPushTokenAsync()`.

## Scope
- Install `expo-notifications` in `mombongo-mobile`
- Create `src/lib/notifications.ts` — `requestPushPermission()` + `registerFcmToken()` helper
- Call from `AuthContext.tsx` after login (same pattern as web)
- Wire `NotificationsScreen.tsx` to a real `useNotifications` hook (call `getNotifications` CF)
- Handle foreground notification display (`setNotificationHandler`)
- Handle notification tap: navigate to the `screen` in `notification.request.content.data`

## Files to create / modify
- `src/lib/notifications.ts` — push token + permission helpers
- `src/context/AuthContext.tsx` — call `registerFcmToken` after login
- `src/hooks/useNotifications.ts` — real hook calling `getNotifications` CF
- `src/screens/NotificationsScreen.tsx` — wire to real hook
- `src/navigation/RootNavigator.tsx` — handle notification tap → navigate

## Implementation

### `src/lib/notifications.ts`
```typescript
import * as Notifications from 'expo-notifications'
import * as Device from 'expo-device'
import { Platform } from 'react-native'
import { httpsCallable } from 'firebase/functions'
import { functions } from './firebase'

const registerFcmTokenFn = httpsCallable<{ token: string }, { success: boolean }>(
  functions, 'registerFcmToken'
)

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
})

export async function requestPushPermission(): Promise<boolean> {
  if (!Device.isDevice) return false // simulators can't receive push
  const { status: existing } = await Notifications.getPermissionsAsync()
  const finalStatus = existing === 'granted'
    ? existing
    : (await Notifications.requestPermissionsAsync()).status
  if (finalStatus !== 'granted') return false

  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('default', {
      name: 'Mombongo',
      importance: Notifications.AndroidImportance.HIGH,
      vibrationPattern: [0, 250, 250, 250],
    })
  }
  return true
}

export async function registerFcmToken(): Promise<void> {
  // Use raw device token so our existing sendPush CF can target it directly via FCM API
  const { data: token } = await Notifications.getDevicePushTokenAsync()
  if (token) await registerFcmTokenFn({ token })
}
```

### AuthContext.tsx — add after login
```typescript
import { requestPushPermission, registerFcmToken } from '@/lib/notifications'

// After successful sign-in:
const granted = await requestPushPermission()
if (granted) await registerFcmToken()
```

### RootNavigator.tsx — notification tap handler
```typescript
const SCREEN_ROUTES: Record<string, string> = {
  market: 'Market',
  financement: 'Financement',
  bourse: 'Bourse',
  exploitation: 'Exploitation',
  academia: 'Academia',
}

useEffect(() => {
  const sub = Notifications.addNotificationResponseReceivedListener(response => {
    const data = response.notification.request.content.data ?? {}
    const screen = SCREEN_ROUTES[data.screen as string]
    if (screen) navigation.navigate(screen as never)
  })
  return () => sub.remove()
}, [navigation])
```

### `src/hooks/useNotifications.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { MOCK_NOTIFICATIONS } from '@/data/mock'
import { isDevMode } from '@/lib/utils'

export function useNotifications() {
  return useQuery({
    queryKey: ['notifications'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_NOTIFICATIONS
      const result = await httpsCallable<void, { notifications: Notification[] }>(
        functions, 'getNotifications'
      )()
      return result.data.notifications
    },
  })
}
```

## Cloud Functions required
- `registerFcmToken` — already exists (deployed)
- `getNotifications` — already exists (SG-08)

## Acceptance criteria
- [ ] On physical Android device: push permission prompt appears after login
- [ ] After granting permission, `users/{uid}.fcmToken` updated in Firestore
- [ ] Morning push sent from `sendMorningPricePush` CF arrives on device
- [ ] Tapping push navigates to `/market` (or relevant screen)
- [ ] NotificationsScreen shows real notifications from `getNotifications` CF
- [ ] On simulator: no crash, no permission prompt (Device.isDevice guard)

## Smoke test
1. Install dev build on physical Android device
2. Sign in as farmer — push permission sheet appears
3. Tap "Allow" — open Firebase console, verify `users/{uid}.fcmToken` updated
4. From Firebase console → Cloud Messaging → send a test message with data `{ "screen": "market", "crop": "Maïs", "province": "Kinshasa" }`
5. Verify notification appears in Android notification shade
6. Tap notification — verify app opens and navigates to Market tab

## Install command
```bash
cd mombongo-mobile
npx expo install expo-notifications expo-device
```
