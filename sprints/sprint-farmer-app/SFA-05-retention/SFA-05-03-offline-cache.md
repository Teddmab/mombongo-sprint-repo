# SFA-05-03 — Offline Cache + Error States (Farmer App)

## Context
Port of SM-09. DRC connectivity is unreliable. Farmers in rural areas may open the app with no connection (after receiving a push on 2G). Without an offline cache, every screen shows an error spinner. With `@tanstack/react-query` + `AsyncStorage` persister, the last-known data is shown even offline.

## Scope
- Install `@react-native-async-storage/async-storage` and `@tanstack/query-async-storage-persister`
- Create a persisted `QueryClient` that survives app restarts
- Set `gcTime: 24 * 60 * 60_000` (24h) on key queries: home, market prices, exploitations, academia modules
- Add connection-aware stale-time adjustment (like web `ConnectionAdaptor`)
- Replace crash-on-no-connection with graceful "Données de votre dernière connexion" banner

## Files to modify
- `App.tsx` — persisted QueryClient setup
- `src/hooks/useNetworkInfo.ts` — detect connection type
- `src/components/OfflineBanner.tsx` — subtle banner when offline showing stale data

## Implementation

### App.tsx — persisted QueryClient
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage'
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister'
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 24 * 60 * 60_000, // keep cache for 24h
      staleTime: 5 * 60_000,
      retry: (failureCount, error) => {
        // Don't retry on network errors when offline — use cache
        if (isNetworkError(error)) return false
        return failureCount < 2
      },
    },
  },
})

const persister = createAsyncStoragePersister({ storage: AsyncStorage })

export default function App() {
  return (
    <PersistQueryClientProvider client={queryClient} persistOptions={{ persister }}>
      {/* app content */}
    </PersistQueryClientProvider>
  )
}
```

### `src/hooks/useNetworkInfo.ts`
```typescript
import NetInfo from '@react-native-community/netinfo'

export function useNetworkInfo() {
  const [isConnected, setConnected] = useState(true)
  const [connectionType, setType] = useState<string | null>('wifi')

  useEffect(() => {
    const unsub = NetInfo.addEventListener(state => {
      setConnected(state.isConnected ?? true)
      setType(state.type)
    })
    return unsub
  }, [])

  return { isConnected, isSlow: connectionType === '2g' || connectionType === 'cellular' }
}
```

### OfflineBanner.tsx
```typescript
export function OfflineBanner() {
  const { isConnected } = useNetworkInfo()
  if (isConnected) return null
  return (
    <View style={styles.banner}>
      <Text style={styles.text}>📵 Données de votre dernière connexion</Text>
    </View>
  )
}
```

## Install commands
```bash
npx expo install @react-native-async-storage/async-storage @react-native-community/netinfo
npm install @tanstack/query-async-storage-persister @tanstack/react-query-persist-client
```

## Acceptance criteria
- [ ] App shows last-known data when opened without internet (after first load)
- [ ] OfflineBanner appears at top of screen when offline
- [ ] Cached data survives app restart (kills app, disables wifi, relaunches)
- [ ] When connection restores, data refreshes automatically
- [ ] Slow connection: staleTime auto-increases to 20 min (less aggressive refetch)

## Smoke test
1. Open app with internet — load home, market, and academia screens
2. Kill app, disable wifi, relaunch
3. Confirm home, market, and academia all show previous data (not error screen)
4. Confirm OfflineBanner visible at top
5. Re-enable wifi — confirm banner disappears and data refreshes
