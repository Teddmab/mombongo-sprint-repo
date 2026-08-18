# SU-01-01 — Morning price push notification

**Sprint:** SU-01 · Daily engagement  
**Branch:** `feature/su-01-daily-engagement`  
**Effort:** ~3 days (1 day CF + 1 day FCM token wiring + 1 day smoke test)

## Context
The single highest-retention action available: every morning at 06:30 WAT, each farmer receives a push notification showing today's price for their primary crop in their province, with a ±% vs yesterday. This gives them a reason to open the app every morning without any further UI change.

## Scope

### Cloud Function (mombongo-functions)
New scheduled CF: `sendMorningPricePush`  
- Trigger: `pubsub.schedule('30 6 * * *').timeZone('Africa/Kinshasa')`
- Logic:
  1. Fetch all users where `role == 'farmer'` and `fcmToken` is set
  2. For each user, read their `primaryCrop` and `province` from their exploitation document
  3. Look up today's price from `province_prices` collection (commodity + province)
  4. Look up yesterday's price (or last known) and compute `delta%`
  5. Send FCM push via `admin.messaging().send()` with:
     - `title`: "Prix du marché ce matin"
     - `body`: `"${crop} à ${price} FC/kg en ${province} ${deltaArrow}${delta}%"`
     - `data`: `{ screen: 'market', crop, province }`
  6. On notification tap: app opens to Prix Marché screen, pre-filtered on that crop

### Frontend (mombongo-web)
- Store FCM token: new CF `saveFcmToken(token: string)` called from `src/store/AuthContext.tsx` on auth state change (after `onAuthStateChanged` resolves with a logged-in user)
- Deep link: notification `data.screen === 'market'` → navigate to `/market?crop=Maïs`
- No UI changes to the dashboard required for this story

## Data model
```
users/{uid}/
  fcmToken: string          ← saved by saveFcmToken CF
  notificationPrefs: {
    morningPrice: boolean   ← default true, user can toggle in Profile
  }

province_prices/{docId}/
  commodity: string
  province: string
  priceUsd: number
  pricePerKgCdf: number     ← add this field if missing
  updatedAt: Timestamp
```

## Acceptance criteria
- [ ] `sendMorningPricePush` CF deploys without error
- [ ] CF fires at 06:30 WAT (verify via Firebase Console → Functions → Logs)
- [ ] A test user with `fcmToken` set receives push within 1 minute of manual trigger
- [ ] Notification body contains correct crop name, price, and delta
- [ ] Tapping notification opens app to Prix Marché, correct crop visible
- [ ] Users with `notificationPrefs.morningPrice = false` are skipped
- [ ] CF handles missing `province_prices` document gracefully (skip user, log warning)

## Smoke test steps
1. In Firebase Console, manually trigger `sendMorningPricePush` (test button in Functions)
2. Verify logs show: user count fetched, prices looked up, FCM sends queued
3. Check test device — notification appears within ~60 seconds
4. Tap notification → app opens → Prix Marché screen shows, filtered to the crop
5. Check that a user with `morningPrice: false` did NOT receive the push

## Implementation notes
- FCM token must be saved AFTER the user is confirmed logged in — not during signup flow
- If `province_prices` has no document for a (crop, province) pair, skip silently and log
- Use `admin.messaging().sendEachForMulticast()` for batch sends (up to 500 tokens per call)
- For DRC timezone: `Africa/Kinshasa` = UTC+1 (no DST)
