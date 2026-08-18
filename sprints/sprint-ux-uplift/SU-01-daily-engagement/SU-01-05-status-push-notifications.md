# SU-01-05 — Status push notifications (credit + listing)

**Sprint:** SU-01 · Daily engagement  
**Branch:** `feature/su-01-daily-engagement`  
**Effort:** ~2 days

## Context
The app collects data but never closes the loop. When a financing application changes status, or when an acheteur views a farmer's Bourse listing, the farmer has no way to know without opening the app. This story adds triggered FCM pushes from Cloud Functions for these two high-value events.

## Events to notify

### Event 1: Financing application status change
Trigger: write to `financingApplications/{appId}` where `status` changes  
Push payload:
- `submitted → under_review`: "Votre dossier est en cours d'examen — décision dans 48h"
- `under_review → approved`: "🎉 Votre crédit est approuvé ! Consultez les détails"
- `under_review → rejected`: "Votre dossier a été examiné — consultez les retours"
- `approved → disbursed`: "Votre crédit a été décaissé sur votre compte"
- Tap: opens `/financement` deep link

### Event 2: Bourse listing receives first view
Trigger: on increment of `listings/{listingId}.viewCount` from 0 to 1  
Push payload: "Un acheteur a regardé votre annonce de [commodity] !"
- Tap: opens `/bourse` filtered to that listing

### Event 3: Agent submits a visit report for this farmer
Trigger: write to `agentReports/{reportId}` where `farmerId` matches  
Push payload: "Votre agent a soumis un rapport de visite — consultez ses recommandations"
- Tap: opens `/exploitation`

## Implementation

### Cloud Functions (mombongo-functions)
- `onFinancingStatusChange`: Firestore trigger `onUpdate` on `financingApplications/{appId}`
- `onListingFirstView`: Firestore trigger `onUpdate` on `listings/{listingId}` (check `before.viewCount === 0 && after.viewCount === 1`)
- `onAgentReportCreated`: Firestore trigger `onCreate` on `agentReports/{reportId}`

Each CF:
1. Reads `farmerId` / `userId` from the document
2. Fetches `users/{uid}.fcmToken` and `notificationPrefs`
3. Calls `admin.messaging().send()` with correct title/body/data
4. Writes to `notifications/{uid}` subcollection (for in-app notification history)

### Frontend
- No new UI needed for the push itself — the existing notification system handles the deep link
- Update notification history rendering to show financing + listing events (if `NotificationsScreen` exists)

## Acceptance criteria
- [ ] Changing `financingApplications/{id}.status` in Firestore Console triggers push to farmer device
- [ ] Incrementing `listings/{id}.viewCount` from 0→1 triggers "acheteur" push
- [ ] `agentReports` creation triggers farmer push
- [ ] Notification tapping opens the correct screen (deep link works)
- [ ] Users with `notificationPrefs.statusUpdates = false` are skipped
- [ ] Each event is written to `notifications/{uid}` subcollection for history
- [ ] No duplicate pushes (idempotent CF — check if notification already sent for this state change)

## Smoke test steps
1. In Firestore Console, change a financing application status from `submitted` to `under_review`
2. Verify push appears on test device within 30 seconds
3. Tap push → verify app opens to `/financement`
4. Set `listings/{id}.viewCount` from 0 to 1 in Firestore Console → verify "acheteur" push
5. Create a new `agentReports` document → verify farmer push
6. Set `notificationPrefs.statusUpdates = false` → repeat steps → verify no push sent
