# SAG-00-02 — Auth Screen Role Lock (Agent App)

## Context
Same as SFA-00-02 but for the Agent App. Agents are staff — they are created by the Mombongo admin team, not by self-signup. The Agent App auth screen should therefore:
1. Show ONLY the login form (no signup / no role selector)
2. After login, verify `userProfile.role === 'agent'` — reject other roles
3. Show a support contact if login fails ("Contactez votre superviseur Mombongo")

## Scope
- In the Agent App build: hide signup tab entirely
- Hide role selector
- Post-login role guard: reject non-agent accounts
- Error message references supervisor contact

## Files to modify
- `src/screens/AuthScreen.tsx` — hide signup for agent build
- `src/context/AuthContext.tsx` — same guard added in SFA-00-02 (both guards go in the same diff)

## Implementation

### AuthScreen.tsx — agent additions
```typescript
import { isAgentApp } from '@/constants/appRole'

// Hide signup tab for agent build — agents are created by admin
{!isAgentApp && (
  <TabSwitcher tabs={['Se connecter', "S'inscrire"]} />
)}

// Different error message for agents
const errorMessage = isAgentApp
  ? 'Identifiants incorrects. Contactez votre superviseur Mombongo.'
  : 'Email ou mot de passe incorrect.'
```

## Acceptance criteria
- [ ] Agent App shows only login form (no "S'inscrire" tab)
- [ ] Logging in with a farmer/investor account shows supervisor contact message
- [ ] Agent login with valid agent credentials succeeds
- [ ] Multi build: signup tab still visible as before

## Smoke test
1. `EXPO_PUBLIC_APP_ROLE=agent npx expo start`
2. Confirm auth screen shows only login form (no signup tab)
3. Login with investor account → see "Contactez votre superviseur" message
4. Login with valid agent account → navigate to agent home
