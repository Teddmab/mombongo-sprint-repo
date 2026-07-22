# SM-09-01 — Image caching + skeleton loaders

**Sprint:** SM-09 · Offline & Performance  
**Branch:** `feature/sm-09-offline-performance`

## Context
`<Image>` components use Expo's default Image which does not cache aggressively. Product thumbnails, farmer avatars, and course artwork reload on every scroll. Also, loading states show blank space instead of skeleton placeholders, creating jarring layout shifts.

## Acceptance criteria
- [ ] Replace `<Image>` from `react-native` with `<Image>` from `expo-image` throughout the codebase
- [ ] `expo-image` caches images to disk with `cachePolicy="disk"` (persists across app restarts)
- [ ] `ProductCard`, `FarmerCard`, `CourseCard`, `BourseCard` all use expo-image
- [ ] Each card has a skeleton state: gray placeholder matching the final layout (no flash of unstyled content)
- [ ] Skeleton uses `Animated.loop + Animated.timing` for a pulse shimmer effect (no external lib needed)
- [ ] `SkeletonCard` component added to `components/ui/SkeletonCard.tsx` — reusable with configurable height/width
- [ ] Lists show 4–6 skeletons while `isLoading` is true
- [ ] `isLoading` passed down to list components from `useProducts`, `useInvestments`, etc.

## Implementation notes
- `expo-image` is in Expo SDK 50+ — verify SDK version in `package.json` before adding
- If SDK < 50, use `react-native-fast-image` as an alternative (requires EAS build)
- Shimmer: white overlay animated from left to right across the gray placeholder
