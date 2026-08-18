# SFA-05-01 — Status Push Notifications (Farmer App)

## Context
Port of web sprint SU-01-05. Firestore-triggered CFs fire pushes when a farmer's financing application changes status (submitted → approved, approved → disbursed, etc.) or when a buyer contacts them on the Bourse. Mobile just needs the FCM token registered (done in SFA-00-04) and the notification tap handler wired to the right screen (done in SFA-00-04's root navigator handler).

This sprint covers:
1. Verifying status pushes arrive and navigate correctly on device
2. Adding an in-app notification badge count (unread count from `getNotifications` CF)
3. Adding a "Marquer tout comme lu" action in NotificationsScreen

## Scope
- Wire notification badge count to tab bar (unread from `useNotifications`)
- Add `markNotificationsRead` CF call in NotificationsScreen on mount
- Test status push flows end-to-end

## Cloud Functions required (already deployed)
- `getNotifications` — list notifications with `isRead` flag
- `markNotificationsRead` — marks all as read, returns new unread count

## Files to modify
- `src/screens/NotificationsScreen.tsx` — wire to real `useNotifications`, add mark-all-read
- `src/navigation/FarmerTabNavigator.tsx` — show badge on Notifications tab

## Implementation

### NotificationsScreen.tsx
```typescript
const { data: notifications } = useNotifications()
const { mutate: markRead } = useMutation({
  mutationFn: () => httpsCallable(functions, 'markNotificationsRead')({}),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['notifications'] }),
})

// On screen mount, mark all as read
useEffect(() => { markRead() }, [])

// Render notification list with type icons:
// financing_status → 💰 icon
// buyer_interest → 🤝 icon
// morning_price → 📊 icon
// streak_reminder → 🔥 icon
```

### FarmerTabNavigator.tsx — badge
```typescript
const { data: notifications } = useNotifications()
const unreadCount = notifications?.filter(n => !n.isRead).length ?? 0

// On Notifications tab:
tabBarBadge: unreadCount > 0 ? unreadCount : undefined
```

## Acceptance criteria
- [ ] Financing status change (via admin panel) triggers push on device within 30s
- [ ] Push tap navigates to Financement tab and highlights the relevant application
- [ ] Notifications tab badge shows unread count
- [ ] Opening Notifications tab clears the badge (marks all read)
- [ ] NotificationsScreen shows real notifications from CF, not mock

## Smoke test
1. Install on physical device, sign in as farmer
2. In Firebase console → Functions → trigger `updateFinancingStatus` for the farmer's application
3. Confirm push arrives within 30s on device
4. Tap push — confirm navigates to Financement tab
5. Open Notifications tab — confirm notification listed, badge clears after opening
