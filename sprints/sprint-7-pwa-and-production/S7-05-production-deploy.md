# S7-05 — Production — Monitoring & Post-Launch

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S7-05 |
| Sprint | Sprint 7 — PWA & Production |
| Branch | `feature/s7-05-monitoring` |
| Merges into | `main` |
| Estimate | 2h |
| Dependencies | S7-04 (production deployed) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Sentry error tracking, Firebase Performance, Firebase Analytics |
| `mombongo-functions` | 🔨 Active | Sentry for Cloud Functions |
| `mombongo-admin` | ✅ Done | — |

---

## mombongo-web

### Step 1 — Sentry

```bash
npm install @sentry/react @sentry/vite-plugin
```

`src/lib/sentry.ts`:

```typescript
import * as Sentry from '@sentry/react'

export function initSentry() {
  if (import.meta.env.PROD) {
    Sentry.init({
      dsn: import.meta.env.VITE_SENTRY_DSN,
      integrations: [
        Sentry.browserTracingIntegration(),
        Sentry.replayIntegration({ maskAllText: false, blockAllMedia: false }),
      ],
      tracesSampleRate:       0.1,   // 10% of transactions
      replaysSessionSampleRate: 0.05,
      replaysOnErrorSampleRate: 1.0, // 100% on error
      environment: import.meta.env.MODE,
    })
  }
}
```

In `src/main.tsx`:
```typescript
import { initSentry } from '@/lib/sentry'
initSentry()
```

Wrap router in `Sentry.ErrorBoundary`:
```tsx
<Sentry.ErrorBoundary
  fallback={<ErrorScreen />}
  onError={(err, info) => console.error('Sentry caught:', err, info)}
>
  <RouterProvider router={router} />
</Sentry.ErrorBoundary>
```

In `vite.config.ts`, add Sentry source maps upload:
```typescript
import { sentryVitePlugin } from '@sentry/vite-plugin'

plugins: [
  // ... existing plugins
  sentryVitePlugin({
    org:       process.env.SENTRY_ORG,
    project:   process.env.SENTRY_PROJECT,
    authToken: process.env.SENTRY_AUTH_TOKEN,
  }),
]
```

Add `VITE_SENTRY_DSN=...` to `.env.production`.

### Step 2 — Firebase Performance Monitoring

```typescript
// src/lib/firebase.ts — add:
import { getPerformance } from 'firebase/performance'
export const perf = getPerformance(app)
```

Firebase Performance auto-instruments:
- Page load time
- First contentful paint
- Firebase SDK calls (Firestore reads, Auth calls)

Add a custom trace for the investment flow:

```typescript
import { trace } from 'firebase/performance'
import { perf } from '@/lib/firebase'

// In InvestModal submit handler:
const investTrace = trace(perf, 'investment_flow')
investTrace.start()
try {
  await investmentService.createInvestment(payload)
} finally {
  investTrace.stop()
}
```

### Step 3 — Firebase Analytics

```typescript
// src/lib/firebase.ts — add:
import { getAnalytics, logEvent } from 'firebase/analytics'
export const analytics = getAnalytics(app)
export { logEvent }
```

Log key conversion events:

```typescript
// On successful investment
logEvent(analytics, 'investment_created', {
  product_id:   productId,
  amount_usd:   amountUsd,
  currency:     'USD',
})

// On course enrollment
logEvent(analytics, 'course_enrolled', { course_id: courseId })

// On deposit initiated
logEvent(analytics, 'deposit_initiated', { amount_usd: amount, currency })

// On wallet deposit confirmed (after PawaPay webhook or Stripe webhook)
logEvent(analytics, 'deposit_completed', { amount_usd: amount })
```

### Step 4 — Error screen

Create `src/pages/ErrorScreen.tsx`:

```tsx
export function ErrorScreen({ error }: { error?: Error }) {
  return (
    <div className="min-h-screen flex flex-col items-center justify-center p-8 text-center">
      <span className="text-6xl mb-4">⚠️</span>
      <h1 className="text-xl font-bold mb-2">{t('error.title')}</h1>
      <p className="text-gray-500 mb-6">{t('error.description')}</p>
      <button
        onClick={() => window.location.reload()}
        className="bg-green-600 text-white px-6 py-3 rounded-2xl font-bold"
      >
        {t('error.reload')}
      </button>
      {import.meta.env.DEV && error && (
        <pre className="mt-6 text-xs text-left bg-red-50 p-4 rounded-xl overflow-auto max-w-full">
          {error.message}
        </pre>
      )}
    </div>
  )
}
```

### Step 5 — i18n keys

```
error.title       → "Une erreur est survenue" / "Something went wrong"
error.description → "Rechargez la page ou revenez plus tard." / "Reload the page or try again later."
error.reload      → "Recharger" / "Reload"
```

---

## mombongo-functions

### Sentry for Cloud Functions

```bash
cd mombongo-functions && npm install @sentry/node
```

In `src/index.ts` (top of file):
```typescript
import * as Sentry from '@sentry/node'
Sentry.init({ dsn: process.env.SENTRY_DSN, tracesSampleRate: 0.1 })
```

Wrap onCall handlers:
```typescript
export const createInvestment = functions.https.onCall(async (data, context) => {
  return await Sentry.withScope(async () => {
    // ... existing implementation
  })
})
```

Set `SENTRY_DSN` as a Firebase secret:
```bash
firebase functions:secrets:set SENTRY_DSN
```

---

## Post-Launch Runbook

After first production deploy, verify:

1. **Auth** — sign up with a real email, verify Firebase Auth console shows the user
2. **Firestore** — create a test investment in dev mode, verify document appears in Firestore console
3. **PawaPay** — test with PawaPay sandbox, trigger STK push on a test number, verify `pawapayWebhook` fires and wallet credits
   **Stripe** — run a test card payment (`4242 4242 4242 4242`), verify `stripeWebhook` fires and wallet credits
4. **FCM** — trigger a test investment, verify push notification appears on a real device
5. **Analytics** — check Firebase Analytics Realtime view shows `investment_created` event
6. **Sentry** — trigger a test error in staging, verify it appears in Sentry dashboard
7. **Lighthouse** — run `lighthouse https://app.mombongo.com`, target scores: Performance ≥ 85, Accessibility ≥ 90, PWA ≥ 90

---

## ✅ Definition of Done
- [ ] Sentry initialized in production, test error captured
- [ ] Firebase Performance traces visible in Firebase console
- [ ] Analytics events logged for investment, enrollment, deposit
- [ ] Error boundary shows `ErrorScreen` on unhandled exceptions
- [ ] Post-launch runbook completed for all 7 checks
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s7-05): monitoring — Sentry, Firebase Performance, Analytics"
git push origin feature/s7-05-monitoring
```
