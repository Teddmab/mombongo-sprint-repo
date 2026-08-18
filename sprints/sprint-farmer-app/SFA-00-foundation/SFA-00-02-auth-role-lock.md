# SFA-00-02 — Auth Screen Role Lock (Farmer App)

## Context
The current mobile auth flow shows a role selector (4 cards: Investisseur, Agriculteur, Agent Terrain, Commerçant). In the Farmer App build, the role is known at compile time (`EXPO_PUBLIC_APP_ROLE=farmer`), so we must:
1. Hide the role selector entirely
2. Lock the role to `farmer` in AuthContext after login
3. Only allow farmer accounts to log in (reject others at the screen level)

The auth CF (`signIn`, `createUser`) already sets `role` in the user profile. We just need the mobile auth screen to enforce it.

## Scope
- Modify `AuthScreen.tsx` to hide role selector when `isFarmerApp === true`
- After login, if `userProfile.role !== 'farmer'`, show an error toast and sign out
- On signup, hardcode role to `'farmer'` (no card selection)
- No backend change needed

## Files to modify
- `src/screens/AuthScreen.tsx` — hide role selector, hardcode signup role
- `src/context/AuthContext.tsx` — add post-login role guard

## Implementation

### AuthScreen.tsx
```typescript
import { isFarmerApp } from '@/constants/appRole'

// In role selector section:
{!isFarmerApp && (
  <RoleSelectorCards value={selectedRole} onChange={setSelectedRole} />
)}

// In signup payload:
const roleToSubmit = isFarmerApp ? 'farmer' : selectedRole
```

### AuthContext.tsx — post-login role guard
```typescript
import { isFarmerApp, isAgentApp } from '@/constants/appRole'

// After userProfile is loaded:
if (isFarmerApp && userProfile.role !== 'farmer') {
  await auth.signOut()
  throw new Error('Ce compte n\'est pas un compte Agriculteur.')
}
if (isAgentApp && userProfile.role !== 'agent') {
  await auth.signOut()
  throw new Error('Ce compte n\'est pas un compte Agent Terrain.')
}
```

## Acceptance criteria
- [ ] Farmer App build: role selector cards are not rendered
- [ ] Farmer App build: signup creates a `role: 'farmer'` user
- [ ] Farmer App build: logging in with an investor account shows "Ce compte n'est pas un compte Agriculteur." and stays on auth screen
- [ ] Multi build (development): role selector still works as before

## Smoke test
1. Run `EXPO_PUBLIC_APP_ROLE=farmer npx expo start`
2. Open auth screen — confirm no role selector cards appear
3. Sign up with a new account — confirm Firestore `users/{uid}.role === 'farmer'`
4. Try logging in with an existing investor account — confirm error toast and no navigation
