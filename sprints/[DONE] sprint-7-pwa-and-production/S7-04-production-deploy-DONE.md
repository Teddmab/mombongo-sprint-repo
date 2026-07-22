# S7-04 — Production — Deploy Checklist & Final Config

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S7-04 |
| Sprint | Sprint 7 — PWA & Production |
| Branch | `feature/s7-04-production-deploy` |
| Merges into | `main` |
| Estimate | 2h |
| Dependencies | S7-03 (all tests passing) |

## Status update (2026-07-16)

**CI/CD pipeline is already in place via SP-07 (GitHub Actions):**

| Item | Status |
|------|--------|
| `.github/workflows/production.yml` — 9-job pipeline | ✅ Done |
| Approval gate (manual reviewers) | ✅ Done |
| Firebase deploy (Firestore rules + indexes) | ✅ Done |
| Cloudflare Pages deploy via `cloudflare/pages-action@v1` | ✅ Done |
| Smoke tests + health check jobs | ✅ Done |
| GitHub Release job | ✅ Done |
| Firebase web config hardcoded (not secrets) | ✅ Done |

**What still remains in S7-04:**
- Validate all Firestore compound indexes are present in `firestore.indexes.json`
- Production env vars review (ensure no secrets in code)
- Admin deploy pipeline

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | ⚠️ Partial | CI/CD done; Firestore indexes review remaining |
| `mombongo-admin` | 🔨 Active | Firebase Hosting deploy (separate site) |
| `mombongo-functions` | ⚠️ Partial | Finalize Firestore rules + indexes |

---

## mombongo-functions

### Step 1 — Firestore indexes

Ensure `firestore.indexes.json` covers all compound queries used in the app:

```json
{
  "indexes": [
    {
      "collectionGroup": "investments",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "investorId", "order": "ASCENDING" },
        { "fieldPath": "investedAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "bourse_investments",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "investorId", "order": "ASCENDING" },
        { "fieldPath": "investedAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "financing_applications",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "investorId", "order": "ASCENDING" },
        { "fieldPath": "createdAt",  "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "transactions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId",    "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "bourse_prices",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "productName",  "order": "ASCENDING" },
        { "fieldPath": "recordedAt",   "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "enrollments",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId",   "order": "ASCENDING" },
        { "fieldPath": "courseId", "order": "ASCENDING" }
      ]
    }
  ]
}
```

Deploy indexes before deploying rules: `firebase deploy --only firestore:indexes`

### Step 2 — Final Firestore security rules review

Ensure every collection has explicit rules (no implicit `allow: false`). Run the emulator suite:

```bash
firebase emulators:start --only firestore &
npm run test:firestore-rules   # (add this script if not present)
```

### Step 3 — Production environment variables

Ensure `functions/.env.production` has:
```
# PawaPay (mobile money — primary for DRC)
PAWAPAY_API_KEY=...
PAWAPAY_ENVIRONMENT=production
PAWAPAY_WEBHOOK_SECRET=...

# Stripe (Visa / Mastercard)
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

Set via Firebase CLI:
```bash
firebase functions:secrets:set PAWAPAY_API_KEY
firebase functions:secrets:set PAWAPAY_WEBHOOK_SECRET
firebase functions:secrets:set STRIPE_SECRET_KEY
firebase functions:secrets:set STRIPE_WEBHOOK_SECRET
```

---

## mombongo-web

### Step 1 — Environment validation

Add `src/lib/env.ts`:

```typescript
const required = [
  'VITE_FIREBASE_API_KEY',
  'VITE_FIREBASE_AUTH_DOMAIN',
  'VITE_FIREBASE_PROJECT_ID',
  'VITE_FIREBASE_STORAGE_BUCKET',
  'VITE_FIREBASE_MESSAGING_SENDER_ID',
  'VITE_FIREBASE_APP_ID',
  'VITE_FCM_VAPID_KEY',
]

for (const key of required) {
  if (!import.meta.env[key]) {
    throw new Error(`Missing required env var: ${key}`)
  }
}
```

Import in `src/main.tsx` before anything else.

### Step 2 — Firebase Hosting config

`firebase.json` (web target):

```json
{
  "hosting": {
    "target": "web",
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [{ "source": "**", "destination": "/index.html" }],
    "headers": [
      {
        "source": "/service-worker.js",
        "headers": [{ "key": "Cache-Control", "value": "no-cache" }]
      },
      {
        "source": "**/*.@(js|css|woff2)",
        "headers": [{ "key": "Cache-Control", "value": "max-age=31536000, immutable" }]
      }
    ]
  }
}
```

### Step 3 — GitHub Actions CD

Create `.github/workflows/deploy-web.yml`:

```yaml
name: Deploy Web

on:
  push:
    branches: [main]
    paths:    ['mombongo-web/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - working-directory: mombongo-web
        run: npm ci && npm run build
        env:
          VITE_FIREBASE_API_KEY:             ${{ secrets.VITE_FIREBASE_API_KEY }}
          VITE_FIREBASE_AUTH_DOMAIN:         ${{ secrets.VITE_FIREBASE_AUTH_DOMAIN }}
          VITE_FIREBASE_PROJECT_ID:          ${{ secrets.VITE_FIREBASE_PROJECT_ID }}
          VITE_FIREBASE_STORAGE_BUCKET:      ${{ secrets.VITE_FIREBASE_STORAGE_BUCKET }}
          VITE_FIREBASE_MESSAGING_SENDER_ID: ${{ secrets.VITE_FIREBASE_MESSAGING_SENDER_ID }}
          VITE_FIREBASE_APP_ID:              ${{ secrets.VITE_FIREBASE_APP_ID }}
          VITE_FCM_VAPID_KEY:                ${{ secrets.VITE_FCM_VAPID_KEY }}
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken:         ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId:         live
          target:            web
        working-directory: mombongo-web
```

Create `.github/workflows/deploy-functions.yml`:

```yaml
name: Deploy Functions

on:
  push:
    branches: [main]
    paths:    ['mombongo-functions/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - working-directory: mombongo-functions
        run: npm ci && npm run build
      - uses: w9jds/firebase-action@master
        with:
          args: deploy --only functions
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

---

## mombongo-admin

### Firebase Hosting (separate site)

In `.firebaserc` add the admin target:

```json
{
  "projects": { "default": "mombongo-prod" },
  "targets": {
    "mombongo-prod": {
      "hosting": {
        "web":   ["mombongo-web-prod"],
        "admin": ["mombongo-admin-prod"]
      }
    }
  }
}
```

Admin deploy:
```bash
cd mombongo-admin && npm run build
firebase deploy --only hosting:admin
```

---

## ✅ Definition of Done
- [ ] Firestore indexes deployed to production
- [ ] Firestore security rules pass emulator test suite
- [ ] PawaPay and Stripe secrets set via `firebase functions:secrets:set`
- [ ] GitHub Actions deploys web on push to `main`
- [ ] GitHub Actions deploys functions on push to `main`
- [ ] `npm run build` exits 0 for all repos

```bash
git commit -m "feat(s7-04): production deploy config — hosting, CD pipeline, indexes"
git push origin feature/s7-04-production-deploy
```
