# SM-09-00 — React Query AsyncStorage persister

**Sprint:** SM-09 · Offline & Performance  
**Branch:** `feature/sm-09-offline-performance`

## Context
Currently all data is re-fetched on every app open. With poor connectivity in rural DRC, screens flash blank before data loads. React Query has a `persistQueryClient` plugin that stores query cache in AsyncStorage, making the app feel instant on reopen.

## Acceptance criteria
- [ ] `@tanstack/query-async-storage-persister` installed
- [ ] `persistQueryClient({ queryClient, persister })` configured in `lib/queryClient.ts`
- [ ] `buster` key tied to app version (from `expo-constants`) so stale cache is cleared on upgrade
- [ ] Persisted queries: `products`, `investments`, `bourse-opportunities`, `courses` (high-value, slow-changing)
- [ ] Non-persisted queries: `wallet`, `notifications` (user-specific, must be fresh)
- [ ] `staleTime: 5 * 60 * 1000` (5 min) for persisted queries so they don't re-fetch every second
- [ ] On first render with stale cache, data shows immediately while background refetch runs
- [ ] Loading skeletons still show on cache miss (first-ever launch)

## Implementation notes
- Use `@react-native-async-storage/async-storage` (already installed) as the storage backend
- `createAsyncStoragePersister` from `@tanstack/query-async-storage-persister`
- `maxAge: 24 * 60 * 60 * 1000` (1 day) — cache evicts after 24h
- Test: kill app, disable network, reopen — products/bourse should show cached data
