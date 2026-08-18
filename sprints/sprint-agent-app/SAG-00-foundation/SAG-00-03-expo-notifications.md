# SAG-00-03 — Expo Notifications (Agent App)

## Context
Same setup as SFA-00-04 but agents have different notification requirements:
- Agents should have 100% push opt-in (they are staff, not optional users)
- Agent-relevant notifications: new farmer assigned, financing status requires action, field report deadline
- No "morning price push" (that's farmer-only)
- Push permission is requested during first login, not during onboarding (no 4-step onboarding for agents)

## Scope
- Reuse `src/lib/notifications.ts` from SFA-00-04 (same module, shared code)
- In AuthContext: request push permission immediately after agent login
- Add notification tap routing for agent-specific screens:
  - `screen: 'farmer_detail'` + `farmerId` → AgentFarmerDetailScreen
  - `screen: 'pipeline'` → AgentPipelineScreen
  - `screen: 'report_due'` → AgentReportFormScreen

## Files to modify
- `src/context/AuthContext.tsx` — call `registerFcmToken` after agent login
- `src/navigation/AgentTabNavigator.tsx` — add notification tap handler

## Implementation

### AgentTabNavigator.tsx — agent-specific routes
```typescript
const AGENT_SCREEN_ROUTES: Record<string, string> = {
  farmer_detail: 'AgentFarmerDetail',
  pipeline: 'AgentPipeline',
  report_due: 'AgentReportForm',
  notifications: 'Notifications',
}

useEffect(() => {
  const sub = Notifications.addNotificationResponseReceivedListener(response => {
    const data = response.notification.request.content.data ?? {}
    const screen = AGENT_SCREEN_ROUTES[data.screen as string]
    if (!screen) return
    const params = data.farmerId ? { farmerId: data.farmerId } : undefined
    navigation.navigate(screen as never, params as never)
  })
  return () => sub.remove()
}, [navigation])
```

## Push permission in AuthContext (agent build)
```typescript
// After successful login, for agent build, request push without waiting for onboarding
if (isAgentApp) {
  const granted = await requestPushPermission()
  if (granted) await registerFcmToken()
}
```

## Acceptance criteria
- [ ] Push permission requested immediately after agent first login
- [ ] `users/{uid}.fcmToken` updated after permission granted
- [ ] Push with `{ screen: 'farmer_detail', farmerId: 'abc' }` navigates to AgentFarmerDetailScreen with correct farmerId
- [ ] Push with `{ screen: 'pipeline' }` navigates to AgentPipelineScreen

## Smoke test
1. Login as agent — confirm permission prompt appears
2. Grant permission — verify token in Firestore
3. Send test push with `{ "screen": "farmer_detail", "farmerId": "testId" }` via Firebase console
4. Tap push — confirm AgentFarmerDetailScreen opens with correct farmer
