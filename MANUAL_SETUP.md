# Mombongo — Manual Setup Checklist

All steps that require human action in external dashboards. Nothing here should be committed to code.

**Status key:** ❌ Not done · ✅ Already done · ⚠️ Verify

---

## 1. Stripe (required for card payments — S6-01)

### 1a. Get your Stripe keys

1. Go to [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)
2. Copy both:
   - **Publishable key** — starts with `pk_test_…` (test) or `pk_live_…` (production)
   - **Secret key** — starts with `sk_test_…` (test) or `sk_live_…` (production)

> Start with test keys until you're ready for live payments.

### 1b. Register the Stripe webhook

1. Go to [dashboard.stripe.com/webhooks](https://dashboard.stripe.com/webhooks)
2. Click **Add endpoint**
3. Set endpoint URL:
   ```
   https://europe-west1-mombongo-dev.cloudfunctions.net/stripeWebhook
   ```
4. Select event: **`payment_intent.succeeded`** only
5. Click **Add endpoint**
6. Copy the **Signing secret** — starts with `whsec_…`

### 1c. Set Firebase Functions secrets

Run these commands from inside `mombongo-functions/`:

```bash
firebase functions:secrets:set STRIPE_SECRET_KEY
# Paste your secret key when prompted

firebase functions:secrets:set STRIPE_WEBHOOK_SECRET
# Paste your webhook signing secret when prompted
```

Firebase Secrets console: [console.firebase.google.com/project/mombongo-dev/functions](https://console.firebase.google.com/project/mombongo-dev/functions)

---

## 2. Firebase Cloud Messaging — VAPID key (required for web push — S6-02)

### 2a. Get the VAPID key

1. Go to [console.firebase.google.com/project/mombongo-dev/settings/cloudmessaging](https://console.firebase.google.com/project/mombongo-dev/settings/cloudmessaging)
2. Scroll to **Web Push certificates**
3. If no key exists, click **Generate key pair**
4. Copy the **Key pair** (the long base64 string)

### 2b. Add to GitHub — mombongo-web

Go to [github.com/Teddmab/mombongo-web/settings/secrets/actions](https://github.com/Teddmab/mombongo-web/settings/secrets/actions)

Add under **Environments → production**:

| Secret name | Value |
|---|---|
| `STAGING_FIREBASE_VAPID_KEY` | Same VAPID key from step 2a |

> `PROD_FIREBASE_VAPID_KEY` ✅ already exists as a repo secret. `STAGING_FIREBASE_VAPID_KEY` is the missing one for the staging workflow build.

---

## 3. GitHub Secrets — mombongo-web

Go to [github.com/Teddmab/mombongo-web/settings/secrets/actions](https://github.com/Teddmab/mombongo-web/settings/secrets/actions)

Add as **Repository secrets**:

| Secret name | Value | Status |
|---|---|---|
| `STRIPE_PUBLISHABLE_KEY` | Your `pk_test_…` or `pk_live_…` key | ❌ Missing |
| `PROD_FIREBASE_VAPID_KEY` | VAPID key from step 2a | ✅ Already set |
| `CLOUDFLARE_API_TOKEN` | Cloudflare token (mombongo-web) | ✅ Already set |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare account ID | ✅ Already set |
| `FIREBASE_TOKEN_PROD` | Firebase CI token | ✅ Already set |

---

## 4. GitHub Secrets — mombongo-mobile

### 4a. Get an Expo access token

1. Go to [expo.dev/accounts/afrotouch-ou/settings/access-tokens](https://expo.dev/accounts/afrotouch-ou/settings/access-tokens)
2. Click **Create token**
3. Name it `github-actions`, set role to **Robot**
4. Copy the token

### 4b. Add to GitHub

Go to [github.com/Teddmab/mombongo-mobile/settings/secrets/actions](https://github.com/Teddmab/mombongo-mobile/settings/secrets/actions)

Add as **Repository secret**:

| Secret name | Value | Status |
|---|---|---|
| `EXPO_TOKEN` | Token from step 4a | ❌ Missing |

---

## 5. Deploy Firebase Functions (Sprint 6)

After setting the Stripe secrets above, deploy the new S6 functions. Run from inside `mombongo-functions/`:

```bash
firebase deploy --only \
  functions:createStripePaymentIntent,\
  functions:stripeWebhook,\
  functions:onInvestmentCreated,\
  functions:onBourseInvestmentCreated,\
  functions:onDepositCompleted,\
  functions:onFinancingStatusChanged,\
  functions:onHarvestDue
```

Firebase Functions console: [console.firebase.google.com/project/mombongo-dev/functions](https://console.firebase.google.com/project/mombongo-dev/functions)

> Deploy **after** step 1c (Stripe secrets) — the functions will fail at runtime if the secrets aren't set first.

---

## 6. GitHub Secrets — mombongo-admin

Go to [github.com/Teddmab/mombongo-admin/settings/secrets/actions](https://github.com/Teddmab/mombongo-admin/settings/secrets/actions)

| Secret name | Where | Status |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | Environment: production | ✅ Already set |
| `CLOUDFLARE_ACCOUNT_ID` | Environment: production | ✅ Already set |
| `FIREBASE_TOKEN` | Repository secret | ✅ Already set |

> The admin repo secrets are in good shape. No action needed.

---

## 7. Mobile — Google Play Store (when ready for production releases)

This is not blocking. Do this before your first production `eas submit`.

1. Go to [play.google.com/console](https://play.google.com/console) and open your app (or create a new app for `com.mombongo.mobile`)
2. Go to **Setup → API access**
3. Link to a Google Cloud project and create a **Service account**
4. Grant the service account **Release manager** role in Play Console
5. Download the JSON key file
6. Rename it `google-play-service-account.json` and place it in the root of `mombongo-mobile/` (it's gitignored)

EAS submit will pick it up automatically from the path configured in `eas.json`.

---

## 8. PawaPay webhook (if not already done)

If PawaPay deposits or payouts are not confirming, the webhook URLs may not be registered.

Log in to the [PawaPay merchant dashboard](https://dashboard.pawapay.io) and set:

| Webhook type | URL |
|---|---|
| Deposit (collection) | `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayWebhook` |
| Payout (disbursement) | `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayPayoutWebhook` |
| Refund | `https://europe-west1-mombongo-dev.cloudfunctions.net/pawapayRefundWebhook` |

Also ensure `PAWAPAY_API_KEY` is set as a Firebase Functions secret:

```bash
firebase functions:secrets:set PAWAPAY_API_KEY
```

---

## Summary — Actions by urgency

| Priority | Action | Blocks |
|---|---|---|
| 🔴 Now | Step 1 — Stripe keys + webhook | Card deposits |
| 🔴 Now | Step 3 — `STRIPE_PUBLISHABLE_KEY` GitHub secret | Web CI build |
| 🔴 Now | Step 4 — `EXPO_TOKEN` GitHub secret | Mobile CI/CD |
| 🔴 Now | Step 5 — Deploy S6 functions | All S6 features |
| 🟡 Soon | Step 2 — `STAGING_FIREBASE_VAPID_KEY` | Web push notifications in staging |
| 🟢 Later | Step 7 — Google Play service account | Play Store production releases |
