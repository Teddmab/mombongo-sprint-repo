# SAG-00-01 — EAS Build Profiles (Agent App)

## Context
Same architectural approach as SFA-00-01. The Agent App uses `EXPO_PUBLIC_APP_ROLE=agent` set in the EAS build profile. The `app.config.js` already handles agent metadata (added in SFA-00-01 — implement both together).

This story focuses on what's agent-specific: the agent tab bar layout and route guard.

## Scope
- Ensure `eas.json` has `agent` and `agent-preview` profiles (shared implementation with SFA-00-01)
- `app.config.js` uses `configs.agent` (name: "Mombongo Agent", bundle ID: `com.mombongo.agent`)
- Agent tab bar: Home | Farmers | Rapports | Pipeline | Notifications
- No investor/farmer/merchant tabs in agent build

## Agent tab bar
```typescript
// src/navigation/AgentTabNavigator.tsx
const tabs = [
  { name: 'Home',          icon: 'home',      screen: AgentHomeScreen },
  { name: 'Farmers',       icon: 'account-group', screen: AgentFarmerListScreen },
  { name: 'Rapports',      icon: 'clipboard-text', screen: AgentReportHistoryScreen },
  { name: 'Pipeline',      icon: 'kanban',    screen: AgentPipelineScreen },
  { name: 'Notifications', icon: 'bell',      screen: NotificationsScreen },
]
```

## Root navigator guard
```typescript
import { isAgentApp } from '@/constants/appRole'

const initialRoute = isAgentApp ? 'AgentTabs' : /* existing multi-role logic */
```

## Acceptance criteria
- [ ] `eas build --profile agent` produces APK named "Mombongo Agent"
- [ ] Agent build shows 5-tab navigation (no investor/farmer/merchant tabs)
- [ ] Bundle ID is `com.mombongo.agent` (distinct from farmer app)

## Smoke test
1. Run `EXPO_PUBLIC_APP_ROLE=agent npx expo start`
2. Verify tab bar shows 5 agent-specific tabs
3. Verify no "Portfolio" or "Bourse" investor/farmer tabs visible
