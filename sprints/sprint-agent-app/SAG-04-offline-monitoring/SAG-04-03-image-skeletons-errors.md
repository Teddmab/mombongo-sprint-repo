# SAG-04-03 — Image Caching, Skeletons, and Network Error States (Agent App)

## Context
Same as SFA-05-04. Agents visit farms with intermittent connectivity. Farmer avatars, field report photos, and farm images must cache on disk so they remain visible offline. All loading states need skeletons instead of blanks; all CF errors need retry affordances.

The `Skeleton`, `ErrorRetryView`, and `expo-image` changes are implemented once in the shared `src/components/` directory — this story ensures they are applied to the agent-specific screens that SFA-05-04 did not cover.

## Scope
- Apply `expo-image` to all image usages in agent screens: farmer avatars in AgentFarmerListScreen + AgentFarmerDetailScreen, field report photo thumbnails in AgentReportDetailScreen
- Apply `SkeletonCard` to: AgentFarmerListScreen (while loading), AgentPipelineScreen (while loading), AgentVisitPlan section
- Apply `ErrorRetryView` to: AgentFarmerListScreen, AgentPipelineScreen, AgentReportHistoryScreen

`Skeleton.tsx` and `ErrorRetryView.tsx` are created in SFA-05-04 — no duplication needed; import from `@/components`.

## Files to modify
- `src/screens/farmers/AgentFarmerListScreen.tsx`
- `src/screens/farmers/AgentFarmerDetailScreen.tsx`
- `src/screens/pipeline/AgentPipelineScreen.tsx`
- `src/screens/reports/AgentReportHistoryScreen.tsx`
- `src/screens/reports/AgentReportDetailScreen.tsx`
- `src/components/VisitPlanSection.tsx`

## Implementation patterns

### Farmer list with skeleton + error
```typescript
const { data, isLoading, isError, error, refetch } = useAgentFarmers()

if (isLoading) {
  return (
    <FlatList
      data={Array(5).fill(null)}
      renderItem={() => <SkeletonCard />}
      keyExtractor={(_, i) => String(i)}
    />
  )
}

return (
  <ErrorRetryView error={isError ? error as Error : null} hasStaleData={!!data} onRetry={refetch}>
    <FlatList data={data} renderItem={({ item }) => <FarmerCard farmer={item} />} />
  </ErrorRetryView>
)
```

### Farmer avatar with expo-image + cache
```typescript
import { Image } from 'expo-image'

// In FarmerCard and AgentFarmerDetailScreen:
<Image
  source={{ uri: farmer.avatarUrl ?? DEFAULT_AVATAR_URI }}
  style={styles.avatar}
  contentFit="cover"
  cachePolicy="memory-disk"
  placeholder={require('@/assets/avatar-placeholder.png')}
  transition={200}
/>
```

### Pipeline screen skeleton
```typescript
const { data: pipeline, isLoading, isError, error, refetch } = useAgentPipeline()

if (isLoading) {
  return (
    <ScrollView horizontal>
      {['À évaluer', 'En cours', 'Approuvé', 'Décaissé'].map(col => (
        <View key={col} style={styles.column}>
          <Text style={styles.colHeader}>{col}</Text>
          {Array(2).fill(null).map((_, i) => <SkeletonCard key={i} />)}
        </View>
      ))}
    </ScrollView>
  )
}

return (
  <ErrorRetryView error={isError ? error as Error : null} hasStaleData={!!pipeline} onRetry={refetch}>
    <PipelineColumns pipeline={pipeline!} />
  </ErrorRetryView>
)
```

### Report photo thumbnails with expo-image
```typescript
// In AgentReportDetailScreen — photo gallery:
{report.photoUrls?.map(url => (
  <Image
    key={url}
    source={{ uri: url }}
    style={styles.thumbnail}
    contentFit="cover"
    cachePolicy="memory-disk"
    transition={150}
  />
))}
```

## Acceptance criteria
- [ ] Farmer list shows 5 skeleton cards while loading (not blank screen)
- [ ] Pipeline screen shows skeleton columns while loading
- [ ] Farmer avatars load from disk cache on second view (no network request)
- [ ] Field report photo thumbnails cached after first load
- [ ] CF error on farmer list shows "Réessayer" button
- [ ] CF error on pipeline with stale data shows orange banner + stale columns

## Smoke test
1. Open agent app on slow connection → farmer list shows pulsing skeletons
2. Disconnect internet mid-load → pipeline shows error retry button
3. Load farmer detail → view avatars → disconnect → reopen detail → avatars still visible from cache
4. Force pipeline error → stale data banner visible with retry link → tap retry → data refreshes
