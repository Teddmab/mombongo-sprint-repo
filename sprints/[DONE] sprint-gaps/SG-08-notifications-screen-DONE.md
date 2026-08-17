# SG-08 — Notifications Screen (Real Implementation)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SG-08 |
| Sprint | Sprint Gaps 08 |
| Branch | `feature/sg-08-notifications-screen` |
| Merges into | `dev` |
| Estimate | 5h |
| Dependencies | S6-02 (push notifications infra) |

---

## Context

`NotificationsScreen.tsx` is a single-line empty div — there is no UI, no data, nothing. Every
role has a "Notifications" nav item that links to this empty screen. S6-02 sent push notifications
via FCM, but the in-app notification list was never built.

---

## Firestore Collection

`notifications/{id}`:
```
{
  userId: string
  type: 'investment_update' | 'match_found' | 'contract_signed' | 'escrow_funded'
       | 'delivery_confirmed' | 'kyc_approved' | 'kyc_rejected' | 'payment_received'
       | 'report_submitted' | 'system'
  title: string
  body: string
  read: boolean
  data: Record<string, string>   // e.g. { contractId, matchId, productId }
  createdAt: Timestamp
}
```

Existing S6-02 FCM sends already write to this collection (if not, update them to do so). All
notification-sending CFs should do:
```typescript
await db.collection('notifications').add({
  userId: recipientUid,
  type,
  title,
  body,
  read: false,
  data: { ...relevantIds },
  createdAt: FieldValue.serverTimestamp(),
})
```

---

## Cloud Functions

### `getMyNotifications`
```typescript
// Params: { limit?: number } (default 50)
// Returns: { notifications: Notification[], unreadCount: number }
export const getMyNotifications = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const limit = data?.limit ?? 50
  const snap = await db.collection('notifications')
    .where('userId', '==', uid)
    .orderBy('createdAt', 'desc')
    .limit(limit)
    .get()

  const notifications = snap.docs.map(d => ({ id: d.id, ...d.data() }))
  const unreadCount = notifications.filter((n: any) => !n.read).length

  return { notifications, unreadCount }
})
```

### `markNotificationsRead`
```typescript
// Params: { notificationIds: string[] } — or pass empty array to mark all read
export const markNotificationsRead = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { notificationIds = [] } = data
  const batch = db.batch()

  if (notificationIds.length === 0) {
    // Mark all unread as read
    const snap = await db.collection('notifications')
      .where('userId', '==', uid)
      .where('read', '==', false)
      .limit(100)
      .get()
    snap.docs.forEach(d => batch.update(d.ref, { read: true }))
  } else {
    notificationIds.forEach((id: string) => {
      batch.update(db.collection('notifications').doc(id), { read: true })
    })
  }

  await batch.commit()
  return { ok: true }
})
```

---

## Web Hook

`src/hooks/useNotifications.ts`:
```typescript
export function useNotifications(limit = 50) {
  if (isDevMode()) return { data: MOCK_NOTIFICATIONS, isLoading: false, unreadCount: 2 }
  return useQuery({
    queryKey: ['notifications', limit],
    queryFn: async () => {
      const fn = httpsCallable(functions, 'getMyNotifications')
      const res = await fn({ limit })
      return res.data as { notifications: Notification[], unreadCount: number }
    },
    staleTime: 30_000,
  })
}

export function useMarkRead() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: async (ids: string[]) => {
      const fn = httpsCallable(functions, 'markNotificationsRead')
      await fn({ notificationIds: ids })
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ['notifications'] }),
  })
}
```

Mock data for dev mode (add to `src/data/mock.ts`):
```typescript
export const MOCK_NOTIFICATIONS = [
  { id: 'n1', type: 'match_found', title: 'Correspondance trouvée', body: 'Un vendeur de Maïs correspond à votre demande.', read: false, createdAt: { seconds: Date.now()/1000 - 3600 }, data: { matchId: 'm1' } },
  { id: 'n2', type: 'kyc_approved', title: 'KYC approuvé', body: 'Votre identité a été vérifiée avec succès.', read: false, createdAt: { seconds: Date.now()/1000 - 86400 }, data: {} },
  { id: 'n3', type: 'payment_received', title: 'Paiement reçu', body: 'Vous avez reçu 50 000 FC de séquestre.', read: true, createdAt: { seconds: Date.now()/1000 - 172800 }, data: { contractId: 'c1' } },
]
```

---

## Web UI

Replace the empty `NotificationsScreen.tsx` with:

```
Notifications                          [Tout marquer lu]

────────────────────────────────────────────────────────
🔵  Correspondance trouvée              il y a 1h
    Un vendeur de Maïs correspond à votre demande.
    [Voir →]
────────────────────────────────────────────────────────
🔵  KYC approuvé                       hier
    Votre identité a été vérifiée avec succès.
────────────────────────────────────────────────────────
    Paiement reçu                      il y a 2j
    Vous avez reçu 50 000 FC de séquestre.
    [Voir →]
────────────────────────────────────────────────────────

Empty state (no notifications):
  🔔 Aucune notification pour l'instant.
```

**Detail**:
- Unread notifications have a blue dot + bold title + slightly highlighted background
- "Tout marquer lu" calls `markNotificationsRead([])` (all)
- Tapping a notification marks it read individually + navigates to relevant screen based on `data`
- Relative timestamps: "il y a 1h", "hier", "il y a 3j"
- `[Voir →]` action button shown when `data` contains a navigable ID

**Nav badge**: The notification icon in the bottom nav / header should show a red badge with `unreadCount` when > 0. Query `unreadCount` from `useNotifications()` in the nav component.

---

## Acceptance Criteria
- [ ] `getMyNotifications` CF returns user's last 50 notifications
- [ ] `markNotificationsRead` marks single or all as read
- [ ] `NotificationsScreen` shows real notification list (not empty div)
- [ ] Unread notifications visually distinct (blue dot, bold, highlight)
- [ ] "Tout marquer lu" button clears all unread
- [ ] Tapping a notification with a `data.contractId` navigates to that contract
- [ ] Relative timestamps ("il y a 1h", "hier")
- [ ] Nav badge shows unread count badge when > 0
- [ ] Empty state shown when no notifications
- [ ] Dev mode: MOCK_NOTIFICATIONS used (2 unread + 1 read)
