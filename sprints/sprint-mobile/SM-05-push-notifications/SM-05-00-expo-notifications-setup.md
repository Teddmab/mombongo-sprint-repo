# SM-05-00 — Push notification permission + FCM token registration

**Sprint:** SM-05 · Push Notifications  
**Branch:** `feature/sm-05-push-notifications`

## Context
Mobile has no push notification setup. Users don't receive notifications for: investment returns, KYC status changes, new bourse opportunities, course unlocks, or agent visit reminders. The web app uses Firebase Cloud Messaging (FCM) from Cloud Functions; mobile should use `expo-notifications` to receive the same FCM messages.

## Acceptance criteria
- [ ] `expo-notifications` installed and configured in `app.json` (android `googleServicesFile`, push notification permissions)
- [ ] `useNotificationPermission()` hook: requests permission on first launch after auth
- [ ] On permission grant, registers for push token via `Notifications.getExpoPushTokenAsync()` or `getDevicePushTokenAsync()`
- [ ] FCM token stored in Firestore via `httpsCallable(functions, "registerPushToken")({ token, platform: "android" | "ios" })`
- [ ] Token refresh handled: `Notifications.addPushTokenListener` calls registerPushToken again
- [ ] Token registration happens in `context/Providers.tsx` after auth state resolves
- [ ] `app.json` `android.googleServicesFile` points to existing `google-services.json`

## Implementation notes
- For Expo managed workflow, use `expo-notifications` (not `@react-native-firebase/messaging`)
- `getExpoPushTokenAsync` requires an Expo project ID — use the one in `eas.json`: `1182d1e2-14eb-4086-9a6e-c1f08291cf86`
- Foreground notification handler: `Notifications.setNotificationHandler` — show alert + badge + sound
- Background: handled by the Expo notifications infrastructure automatically
- Cloud Function: `registerPushToken` stores token in `users/{uid}/pushTokens` array (via arrayUnion)
