# SM-10-00 — Sentry React Native: crash reporting + error boundary

**Sprint:** SM-10 · Monitoring  
**Branch:** `feature/sm-10-monitoring`

## Context
The web app uses `@sentry/react` with error boundaries and session replay. Mobile has no crash reporting — production crashes are invisible. With users in rural DRC on low-end Android devices, crash signals are critical.

## Acceptance criteria
- [ ] `@sentry/react-native` installed
- [ ] `Sentry.init({ dsn: process.env.EXPO_PUBLIC_SENTRY_DSN, environment, tracesSampleRate: 0.3 })` in `app/_layout.tsx` before render
- [ ] `Sentry.ErrorBoundary` wraps the root navigator in `app/_layout.tsx`
- [ ] Error boundary fallback: full-screen error page with "Signaler le problème" + "Relancer l'app"
- [ ] `EXPO_PUBLIC_SENTRY_DSN` added to all `eas.json` build profiles
- [ ] Source maps uploaded via EAS build hook (sentry-expo or `@sentry/react-native` Metro plugin)
- [ ] User context set after auth: `Sentry.setUser({ id: uid, email })`
- [ ] User context cleared on sign-out: `Sentry.setUser(null)`
- [ ] `ci.yml` does not need Sentry changes (source maps only via EAS)

## Implementation notes
- DSN: use the same Sentry project as mombongo-web (or create a separate "mombongo-mobile" project in Sentry)
- `sentry-expo` is the Expo-specific wrapper — check if it's still maintained vs using `@sentry/react-native` directly
- Session replay: skip for MVP (heavy on resources for low-end devices)
- `tracesSampleRate: 0.3` means 30% of sessions have performance traces
