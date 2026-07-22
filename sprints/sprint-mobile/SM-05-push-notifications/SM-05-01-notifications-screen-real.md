# SM-05-01 — NotificationsScreen: real data from Cloud Function

**Sprint:** SM-05 · Push Notifications  
**Branch:** `feature/sm-05-push-notifications`

## Context
`NotificationsScreen` currently reads from `useLocalData.useNotifications()` which stores items in AsyncStorage. This is local-only — notifications are not synced across devices and don't reflect server-sent events.

## Acceptance criteria
- [ ] `hooks/useNotifications.ts` created (separate from `useLocalData`)
- [ ] `useNotifications()` calls `httpsCallable(functions, "getNotifications")` → `{ notifications: Notification[] }`
- [ ] `NotificationsScreen` uses the new hook; removes `useLocalData` import
- [ ] "Marquer tout comme lu" calls `httpsCallable(functions, "markNotificationsRead")({ ids: [...] })`
- [ ] Unread notifications highlighted with a left border or dot indicator
- [ ] `useUnreadCount()` derived hook used in `(tabs)/_layout.tsx` to show badge on the notifications tab icon
- [ ] In devMode, returns `MOCK_NOTIFICATIONS` from `data/mock.ts`
- [ ] Pull-to-refresh refetches the query

## Data shape
```ts
interface Notification {
  id: string;
  kind: "profit" | "opportunity" | "report" | "course" | "system";
  title: string;
  body: string;
  read: boolean;
  createdAt: { seconds: number };
  deepLink?: string;  // e.g. "/bourse/opp123"
}
```

## Cloud Functions required
- `getNotifications()` → `{ notifications: Notification[] }` — last 50, ordered by createdAt desc
- `markNotificationsRead({ ids: string[] })` → sets `read: true` on each

## Deep link handling
- If `deepLink` is set on a received push, navigate to it via `expo-router` `router.push(deepLink)`
- Add `Notifications.addNotificationResponseReceivedListener` in `_layout.tsx`
