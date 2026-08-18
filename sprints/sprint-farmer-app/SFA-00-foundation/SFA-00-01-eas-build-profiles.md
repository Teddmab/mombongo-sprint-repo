# SFA-00-01 — EAS Build Profiles (Farmer App)

## Context
The current `mombongo-mobile` codebase is a single multi-role app. To ship a dedicated Farmer App, we use EAS Build profiles to lock the build to the farmer role via an environment variable read at app startup. This avoids maintaining two separate codebases.

## Scope
- Add `farmer` profile to `eas.json`
- Set `EXPO_PUBLIC_APP_ROLE=farmer` in that profile
- Update `app.config.js` to read `EXPO_PUBLIC_APP_ROLE` and apply role-specific values (name, slug, bundle ID, icon, scheme)
- Add startup guard in `App.tsx` (or `_layout.tsx` for Expo Router) that reads `EXPO_PUBLIC_APP_ROLE` and forces the correct initial route, bypassing the role selector
- No Firestore, no Storage: only read env var + navigation

## Files to create / modify
- `eas.json` — add `farmer` and `agent` profiles
- `app.config.js` — conditional metadata per role
- `src/constants/appRole.ts` — `export const APP_ROLE = process.env.EXPO_PUBLIC_APP_ROLE ?? 'multi'`
- `src/navigation/RootNavigator.tsx` (or equivalent) — guard by `APP_ROLE`

## Implementation

### `eas.json` additions
```json
{
  "build": {
    "farmer": {
      "distribution": "store",
      "android": { "buildType": "apk" },
      "env": {
        "EXPO_PUBLIC_APP_ROLE": "farmer"
      }
    },
    "agent": {
      "distribution": "store",
      "android": { "buildType": "apk" },
      "env": {
        "EXPO_PUBLIC_APP_ROLE": "agent"
      }
    },
    "farmer-preview": {
      "distribution": "internal",
      "env": { "EXPO_PUBLIC_APP_ROLE": "farmer" }
    },
    "agent-preview": {
      "distribution": "internal",
      "env": { "EXPO_PUBLIC_APP_ROLE": "agent" }
    }
  }
}
```

### `app.config.js` role switching
```js
const role = process.env.EXPO_PUBLIC_APP_ROLE ?? 'multi'
const configs = {
  farmer: {
    name: 'Mombongo Agriculteur',
    slug: 'mombongo-farmer',
    bundleIdentifier: 'com.mombongo.farmer',
    package: 'com.mombongo.farmer',
    icon: './assets/icon-farmer.png',
    scheme: 'mombongo-farmer',
  },
  agent: {
    name: 'Mombongo Agent',
    slug: 'mombongo-agent',
    bundleIdentifier: 'com.mombongo.agent',
    package: 'com.mombongo.agent',
    icon: './assets/icon-agent.png',
    scheme: 'mombongo-agent',
  },
  multi: {
    name: 'Mombongo',
    slug: 'mombongo',
    bundleIdentifier: 'com.mombongo.app',
    package: 'com.mombongo.app',
    icon: './assets/icon.png',
    scheme: 'mombongo',
  },
}
module.exports = { expo: { ...existingConfig, ...configs[role] } }
```

### `src/constants/appRole.ts`
```typescript
export type AppRole = 'farmer' | 'agent' | 'multi'
export const APP_ROLE: AppRole =
  (process.env.EXPO_PUBLIC_APP_ROLE as AppRole | undefined) ?? 'multi'
export const isFarmerApp = APP_ROLE === 'farmer'
export const isAgentApp  = APP_ROLE === 'agent'
```

### Startup guard (add to root navigator or `_layout.tsx`)
```typescript
import { APP_ROLE } from '@/constants/appRole'

// Inside navigator setup:
// If role-locked build, skip language + role selector, go straight to farmer tabs
const initialRoute = APP_ROLE === 'farmer' ? 'FarmerTabs'
  : APP_ROLE === 'agent'  ? 'AgentTabs'
  : 'RoleSelect'
```

## Acceptance criteria
- [ ] `eas build --profile farmer` produces an APK whose app name is "Mombongo Agriculteur"
- [ ] `eas build --profile agent` produces an APK whose app name is "Mombongo Agent"
- [ ] Both builds have distinct bundle IDs
- [ ] `isFarmerApp === true` in a farmer build; `isAgentApp === true` in an agent build
- [ ] Multi build (no env var) works as before — shows role selector

## Smoke test
1. Run `EXPO_PUBLIC_APP_ROLE=farmer npx expo start` in Expo Go
2. Verify app header shows "Mombongo Agriculteur" (or check constants file value)
3. Verify auth screen shows no role selector card
4. Verify the tab bar shows only Farmer-relevant tabs (no "Portfolio" investor tab)
