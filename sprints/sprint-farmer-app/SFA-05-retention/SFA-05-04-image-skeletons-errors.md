# SFA-05-04 — Image Caching, Skeletons, and Network Error States (Farmer App)

## Context
Port of SM-09-01 and SM-09-02. The offline persister (SFA-05-03) handles query data. This sprint handles the visual layer: product/farm images load slowly on 2G → need cached images with skeletons; when a CF call fails → need a retry state, not a blank screen.

## Scope

### Image caching
- Replace bare `<Image>` with `expo-image` (has built-in LRU disk cache)
- All farmer home cards, market price icons, exploitation photos, and academy course thumbnails use `expo-image`
- Cache policy: `memory-disk` with 7-day disk TTL

### Skeletons
- Create `src/components/Skeleton.tsx` — animated pulse placeholder
- Use skeleton in: FarmerHomeScreen (while CTA + revenue load), MarketScreen (while prices load), ExploitationList, AcademiaScreen

### Network error states
- Create `src/components/ErrorRetryView.tsx` — shows error message + "Réessayer" button
- Use in any screen where `isError === true` from a useQuery hook
- Include the stale-data fallback: if `isError && data`, show stale data with a banner ("Données pas à jour — Vérifiez votre connexion")

## Files to create
- `src/components/Skeleton.tsx`
- `src/components/ErrorRetryView.tsx`
- Update key screens to use these components

## Implementation

### `src/components/Skeleton.tsx`
```typescript
import Animated, { useSharedValue, useAnimatedStyle, withRepeat, withTiming } from 'react-native-reanimated'

export function Skeleton({ width, height, borderRadius = 8, style }: {
  width: number | string
  height: number
  borderRadius?: number
  style?: object
}) {
  const opacity = useSharedValue(0.4)

  useEffect(() => {
    opacity.value = withRepeat(withTiming(1, { duration: 800 }), -1, true)
  }, [])

  const animatedStyle = useAnimatedStyle(() => ({ opacity: opacity.value }))

  return (
    <Animated.View
      style={[{ width, height, borderRadius, backgroundColor: '#E5E7EB' }, animatedStyle, style]}
    />
  )
}

// Convenience preset for card rows
export function SkeletonCard() {
  return (
    <View style={{ padding: 16, gap: 8 }}>
      <Skeleton width="60%" height={16} />
      <Skeleton width="40%" height={12} />
    </View>
  )
}
```

### `src/components/ErrorRetryView.tsx`
```typescript
type Props = {
  error: Error | null
  hasStaleData?: boolean
  onRetry: () => void
  children?: React.ReactNode
}

export function ErrorRetryView({ error, hasStaleData, onRetry, children }: Props) {
  if (!error) return <>{children}</>

  if (hasStaleData) {
    return (
      <>
        <View style={styles.staleBanner}>
          <Text style={styles.staleText}>⚠️ Données pas à jour · Vérifiez votre connexion</Text>
          <TouchableOpacity onPress={onRetry}>
            <Text style={styles.retryLink}>Réessayer</Text>
          </TouchableOpacity>
        </View>
        {children}
      </>
    )
  }

  return (
    <View style={styles.errorContainer}>
      <Text style={styles.errorTitle}>Impossible de charger les données</Text>
      <Text style={styles.errorSub}>{error.message.includes('network') ? 'Vérifiez votre connexion internet' : 'Une erreur est survenue'}</Text>
      <TouchableOpacity style={styles.retryButton} onPress={onRetry}>
        <Text>Réessayer</Text>
      </TouchableOpacity>
    </View>
  )
}
```

### expo-image replacement
```typescript
// Before:
import { Image } from 'react-native'
<Image source={{ uri: url }} style={...} />

// After:
import { Image } from 'expo-image'
<Image source={{ uri: url }} style={...} contentFit="cover" cachePolicy="memory-disk" />
```

### Usage pattern in screens
```typescript
const { data, isLoading, isError, error, refetch } = useDashboardCTA()

if (isLoading) return <SkeletonCard />

return (
  <ErrorRetryView error={isError ? error as Error : null} hasStaleData={!!data} onRetry={refetch}>
    <DashboardCTACard cta={data!} />
  </ErrorRetryView>
)
```

## Install command
```bash
npx expo install expo-image react-native-reanimated
```

## Acceptance criteria
- [ ] Home, market, exploitation, and academia screens show pulsing skeletons while loading (not blank)
- [ ] Images in cards use `expo-image` with `memory-disk` cache policy
- [ ] Second load of images is instant (from disk cache)
- [ ] When a CF fails: error view with "Réessayer" button appears (not crash or blank)
- [ ] When stale data is available: data shown with orange banner + retry link
- [ ] Skeleton animation respects `prefers-reduced-motion` (stops pulsing if user has reduced motion enabled)

## Smoke test
1. Open app on slow connection — confirm skeletons visible for 1–3s before data loads
2. Force a CF error (disable internet mid-load) — confirm "Réessayer" button appears
3. Tap "Réessayer" with internet restored — data loads correctly
4. Load market screen twice — second load images are instant
5. Open app with no internet (stale data available) — orange banner visible, data still shown
