# SM-09-02 — Network error states + retry UI

**Sprint:** SM-09 · Offline & Performance  
**Branch:** `feature/sm-09-offline-performance`

## Context
When Cloud Functions calls fail (network timeout, offline, CF error), screens currently show blank or crash. This story adds consistent error states with retry.

## Acceptance criteria
- [ ] `ErrorScreen` component (or in-line `ErrorCard`) shows when `isError === true` on any screen's primary query
- [ ] Error card shows: icon + "Problème de connexion" + error message + "Réessayer" button
- [ ] "Réessayer" calls `refetch()` from the query
- [ ] Offline banner: persistent bar at the top when `NetInfo.isConnected === false`
- [ ] `useNetworkStatus()` hook wrapping `@react-native-community/netinfo` (or `expo-network`)
- [ ] When connection restored, in-flight queries automatically retry (React Query's `retryOnMount`)
- [ ] `queryClient` configured with `retry: 2, retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 10000)`
- [ ] CF errors with `code === "unauthenticated"` redirect to auth screen instead of showing retry

## Implementation notes
- `expo-network` is in Expo SDK — use `Network.getNetworkStateAsync()` or subscribe to changes
- Offline banner: fixed `position: absolute` at `top: 0`, `zIndex: 9999`, amber background
- "Réessayer" banner disappears when connection restored + queries succeed
