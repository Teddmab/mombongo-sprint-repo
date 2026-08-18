# SAG-04-02 — Sentry + Analytics + iOS (Agent App)

## Context
Same as SFA-06-01 but scoped to agent-specific events. Also covers iOS EAS build for agent app (same as SFA-06-02 but for `agent` profile).

## Scope
- Sentry React Native setup (shared setup file with farmer app — `src/lib/sentry.ts`)
- Agent-specific analytics events
- iOS EAS profile for agent app

## Agent analytics events

### `src/lib/analytics.ts` — agent additions
```typescript
export const agentTrack = {
  visitCompleted: (p: { farmerId: string; durationMinutes: number }) =>
    Analytics.logEvent('visit_completed', p),
  reportSubmitted: (p: { farmerId: string; photoCount: number; hasPhotos: boolean }) =>
    Analytics.logEvent('report_submitted', p),
  assessmentSubmitted: (p: { farmerId: string; recommendation: 'approve' | 'reject'; score: number }) =>
    Analytics.logEvent('assessment_submitted', p),
  disbursementConfirmed: (p: { farmerId: string; amountCdf: number }) =>
    Analytics.logEvent('disbursement_confirmed', p),
  visitPlanOpened: () =>
    Analytics.logEvent('visit_plan_opened', {}),
  pipelineViewed: (p: { pendingActions: number }) =>
    Analytics.logEvent('pipeline_viewed', p),
}
```

## iOS EAS profile for agent
```json
// eas.json:
"agent-ios": {
  "distribution": "store",
  "ios": { "buildConfiguration": "Release" },
  "env": { "EXPO_PUBLIC_APP_ROLE": "agent" }
},
"agent-ios-preview": {
  "distribution": "internal",
  "ios": { "buildConfiguration": "Debug" },
  "env": { "EXPO_PUBLIC_APP_ROLE": "agent" }
}
```

## Acceptance criteria
- [ ] Sentry catches crashes in agent build (same DSN or separate agent DSN)
- [ ] `visit_completed` event fires when agent marks a visit done
- [ ] `report_submitted` event fires with photo count
- [ ] `assessment_submitted` fires with recommendation
- [ ] iOS agent build installs on iPhone via TestFlight
- [ ] No safe area issues on iOS

## Smoke test
1. Trigger a test crash in agent build → verify in Sentry dashboard
2. Complete a visit → confirm `visit_completed` in Firebase Analytics DebugView
3. Submit a report with 2 photos → confirm `report_submitted` with `photoCount: 2`
4. Install iOS agent build → navigate all tabs → no safe area issues
