# SP-07 — env.ts Production Fix + Service Worker Cache Cleanup

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-07 |
| Sprint | Sprint Patch 07 |
| Branch | `feature/sp-07-fix-env-prod-validation` |
| Merges into | `main` |
| Estimate | 1h |
| Status | DONE — PR #77 merged |

## Root cause
Two separate production bugs found together:

### Bug 1 — env.ts crash on page load
`env.ts:13 Uncaught Error: Missing required env var: VITE_FIREBASE_API_KEY` was thrown on every page load even though Cloudflare Pages had all env vars set.

**Root cause**: Vite does NOT inline `import.meta.env[dynamicKey]` in production builds — it only replaces static dot-notation references (`import.meta.env.VITE_KEY`). The loop `for (const key of required) { if (!import.meta.env[key]) ... }` left every check as `undefined` after bundling.

**Fix**: Replace the dynamic loop with explicit static references for each variable.

```typescript
// Before (broken in production):
const required = ['VITE_FIREBASE_API_KEY', ...] as const
for (const key of required) {
  if (!import.meta.env[key]) throw new Error(...)
}

// After (correct):
if (import.meta.env.PROD) {
  const checks: [string, string | undefined][] = [
    ['VITE_FIREBASE_API_KEY',            import.meta.env.VITE_FIREBASE_API_KEY],
    ['VITE_FIREBASE_AUTH_DOMAIN',        import.meta.env.VITE_FIREBASE_AUTH_DOMAIN],
    ['VITE_FIREBASE_PROJECT_ID',         import.meta.env.VITE_FIREBASE_PROJECT_ID],
    ['VITE_FIREBASE_STORAGE_BUCKET',     import.meta.env.VITE_FIREBASE_STORAGE_BUCKET],
    ['VITE_FIREBASE_MESSAGING_SENDER_ID',import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID],
    ['VITE_FIREBASE_APP_ID',             import.meta.env.VITE_FIREBASE_APP_ID],
  ]
  for (const [key, val] of checks) {
    if (!val) throw new Error(`Missing required env var: ${key}`)
  }
}
```

### Bug 2 — Service worker 404 on old content-hash filenames
After a new build deployment, the service worker's precache manifest referenced old content-hash filenames (e.g. `leaf-B52BYj5p.js`) that no longer existed, causing `bad-precaching-response` 404 errors.

**Fix**: Add `cleanupOutdatedCaches: true` to workbox config in `vite.config.ts` so stale precache entries are purged on service worker activation.

## Files changed
- `src/lib/env.ts` — replace dynamic loop with static checks
- `vite.config.ts` — add `cleanupOutdatedCaches: true` to workbox config

## Acceptance criteria
- [x] Production deploy: no `env.ts:13 Uncaught Error` on page load
- [x] `npm run build` exits 0 with correct env vars
- [x] Hard reload: service worker does not 404 on old content-hash filenames
- [x] `cleanupOutdatedCaches: true` present in workbox config
