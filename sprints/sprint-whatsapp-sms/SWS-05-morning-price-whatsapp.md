# SWS-05 — Morning Price Push via WhatsApp/SMS

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SWS-05 |
| Branch | `feature/sws-05-morning-price-whatsapp` |
| Merges into | `dev` (mombongo-functions) |
| Estimate | 2h |
| Dependencies | SWS-01 (`notifyUser` helper deployed), SU-01-01 (`sendMorningPricePush` deployed) |

## Context
`sendMorningPricePush` currently sends FCM push only — it calls `sendPush(uid, title, body, data)` per farmer. FCM only reaches farmers who have the app installed with permission granted. In DRC, many farmers will get the WhatsApp notification first (near-100% WhatsApp penetration) and only install the app later.

This story adds WhatsApp (with SMS fallback) as a second channel for the morning price message, using the `notifyUser` helper from SWS-01. Farmers who opt out of WhatsApp/SMS (via `notificationPrefs.whatsapp === false`) are skipped, same as FCM opt-out.

**Result:** each eligible farmer receives the morning price via both channels — FCM push (if app installed) AND WhatsApp (if phone number on file).

---

## mombongo-functions

### Step 1 — Refactor `sendMorningPricePush.ts` to use `sendMorningPricePushCore`

This was already outlined in SAN-01. Prerequisite: extract the handler body into `sendMorningPricePushCore()`.

### Step 2 — Add `notifyUser` call inside the fan-out loop

Inside `sendMorningPricePushCore()`, after the existing `sendPush` call per farmer, add a `notifyUser` call with the same message formatted as plain text (WhatsApp/SMS cannot render data payloads, only text).

```typescript
import { notifyUser } from './notifyUser'  // from SWS-01

// Inside the fan-out loop, replace the current sendPush section with:
await Promise.all(
  group.uids.map(async uid => {
    // 1. FCM push (existing — keeps deep-link data payload)
    await sendPush(uid, title, body, data).catch(err =>
      functions.logger.warn(`sendMorningPricePush: FCM failed for ${uid}`, err)
    )

    // 2. WhatsApp / SMS (new — plain text, no deep-link)
    await notifyUser(uid, `📊 ${body}\nOuvrez Mombongo pour voir tous les prix: mombongo.app/market`).catch(err =>
      functions.logger.warn(`sendMorningPricePush: WhatsApp/SMS failed for ${uid}`, err)
    )
  })
)
```

### WhatsApp message template
The message sent via WhatsApp will be plain text (not a template — templates require Meta approval):

```
📊 Maïs à 450 FC/kg en Kinshasa ↑5%
Ouvrez Mombongo pour voir tous les prix: mombongo.app/market
```

**Note on WhatsApp template approval:** For production, Twilio requires pre-approved message templates for outbound WhatsApp Business messages to new recipients. Register a template like:

> Template name: `morning_price_push`
> Template body: `📊 {{1}} à {{2}} FC/kg en {{3}}{{4}}. Ouvrez Mombongo pour voir tous les prix.`
> Variables: commodity, price, province, delta

Until the template is approved, the sandbox will still work for registered test numbers.

### Updated `notifyUser` call with opt-out check
The existing `notifyUser` helper in SWS-01 already checks `notificationPrefs.whatsapp !== false` before sending — no additional guard needed here.

### Optional: dedicated morning price WhatsApp template
If we want to use an approved template instead of freeform:

```typescript
import { sendWhatsApp } from './notificationService'

// After getTemplate approved by Meta:
await sendWhatsApp({
  phone: userPhone,
  message: body, // fallback
  templateSid: 'HXxxxxx',  // approved template SID from Twilio
  templateVars: {
    '1': group.crop,
    '2': pricePerKgCdf.toLocaleString('fr-FR'),
    '3': group.province,
    '4': deltaStr,
  },
})
```

---

## User preference: opt-out from morning WhatsApp price push

The `notificationPrefs` object on `users/{uid}` already supports per-channel opt-out from SWS-02. For the morning price, we add a specific opt-out flag:

```typescript
// Farmer can opt out of morning WhatsApp price push specifically:
// users/{uid}.notificationPrefs.morningPriceWhatsapp = false

// In the fan-out loop:
const prefs = farmerData.notificationPrefs ?? {}
const skipWhatsApp = prefs.whatsapp === false || prefs.morningPriceWhatsapp === false

if (!skipWhatsApp) {
  await notifyUser(uid, whatsappBody).catch(...)
}
```

---

## mombongo-web — notification preferences UI

Add a toggle in the web NotificationsPreferences section (inside ProfileScreen or Settings):

```tsx
<PreferenceRow
  label="Prix du matin par WhatsApp/SMS"
  description="Recevez le prix de votre culture chaque matin à 06h30"
  checked={prefs.morningPriceWhatsapp !== false}
  onChange={(val) => updateNotifPref('morningPriceWhatsapp', val)}
/>
```

---

## Acceptance criteria
- [ ] Farmers with a phone number on file receive a WhatsApp message at 06:30 WAT
- [ ] Message format: `📊 {crop} à {price} FC/kg en {province}{delta}\n[app link]`
- [ ] Farmers who opt out via `notificationPrefs.morningPriceWhatsapp = false` receive no WhatsApp
- [ ] Farmers with no phone number: FCM only (no WhatsApp attempt, no error)
- [ ] WhatsApp failure (Twilio error) does not prevent FCM from sending — each is caught independently
- [ ] `npm run build` exits 0 after the changes

## Smoke test
1. Set a real phone number on a farmer's `users/{uid}.phone` field (Twilio sandbox registered number)
2. From admin panel → trigger morning price push manually (SAN-01)
3. Confirm FCM push arrives on device (if browser tab open)
4. Confirm WhatsApp message arrives on the test phone within 60s
5. Set `notificationPrefs.morningPriceWhatsapp = false` on the user → trigger again
6. Confirm WhatsApp NOT received; FCM still received ✓

## Deploy command
```bash
firebase deploy --only functions:sendMorningPricePush,functions:adminTriggerMorningPricePush
```

## ⚠️ WhatsApp setup prerequisites (manual — confirm with Teddy)
- Twilio WhatsApp Sandbox activated (or approved Business number)
- Test phone number registered in Twilio sandbox (`join <sandbox-word>` sent to +1 415 523 8886)
- OR: Meta-approved template `morning_price_push` registered via Twilio console
- Firebase secrets `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_WHATSAPP_FROM` set (done in SWS-01)
