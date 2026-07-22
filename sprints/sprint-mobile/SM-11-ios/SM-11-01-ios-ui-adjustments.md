# SM-11-01 — iOS UI adjustments: safe area, haptics, font rendering

**Sprint:** SM-11 · iOS  
**Branch:** `feature/sm-11-ios`

## Context
React Native apps often have iOS-specific rendering differences: safe areas, dynamic island conflicts, haptic feedback, and font rendering. After SM-11-00, this story ensures the app looks and feels native on iPhone.

## Acceptance criteria
- [ ] All screens tested on iPhone 15 Pro (dynamic island) — no overlapping content with notch/island
- [ ] `useSafeAreaInsets()` used correctly in all stack screens (not just tab screens)
- [ ] `StackHeader` component: `statusBarStyle="dark"` on iOS light mode, `"light"` on dark backgrounds
- [ ] Haptic feedback added to primary CTAs:
  - "Investir", "Financer", "Payer" → `Haptics.impactAsync(ImpactFeedbackStyle.Medium)`
  - Error states → `Haptics.notificationAsync(NotificationFeedbackType.Error)`
  - Success states → `Haptics.notificationAsync(NotificationFeedbackType.Success)`
- [ ] `expo-haptics` imported from existing Expo SDK (no new install needed)
- [ ] Keyboard avoiding: `KeyboardAvoidingView` with `behavior="padding"` on iOS (already done in some modals — verify all)
- [ ] `ScrollView` with `keyboardDismissMode="on-drag"` on all forms
- [ ] Tab bar on iOS uses `BlurView` background (already handled by expo-router's default tab layout)
- [ ] `<StatusBar style="auto" />` from `expo-status-bar` used consistently

## Test matrix
| Screen | iPhone 15 Pro | iPhone SE (small) | iPhone 14 (no island) |
|--------|--------------|-------------------|----------------------|
| Home | ✓ | ✓ | ✓ |
| Market | ✓ | ✓ | ✓ |
| Bourse | ✓ | ✓ | ✓ |
| Auth | ✓ | ✓ | ✓ |
| Profile | ✓ | ✓ | ✓ |
| Modals | ✓ | ✓ | ✓ |
