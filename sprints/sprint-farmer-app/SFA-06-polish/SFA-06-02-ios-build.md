# SFA-06-02 — iOS EAS Build (Farmer App)

## Context
Port of SM-11. Android is the primary target for DRC (>85% market share) but iOS builds are needed for TestFlight distribution to Teddy and pilot testers with iPhones.

## Scope
- Add iOS EAS profile for `farmer` build
- Fix known iOS-specific UI issues: safe area insets, tap target sizes, font rendering
- Configure APNs push credentials in EAS for iOS push notifications
- Submit to TestFlight via EAS Submit

## EAS configuration
```json
// eas.json additions under "build":
"farmer-ios": {
  "distribution": "store",
  "ios": { "buildConfiguration": "Release" },
  "env": { "EXPO_PUBLIC_APP_ROLE": "farmer" }
},
"farmer-ios-preview": {
  "distribution": "internal",
  "ios": { "buildConfiguration": "Debug" },
  "env": { "EXPO_PUBLIC_APP_ROLE": "farmer" }
}
```

## Common iOS fixes needed
- `SafeAreaView` import from `react-native-safe-area-context` (not `react-native`) — check all screens
- Bottom tab bar: add `paddingBottom` from `useSafeAreaInsets().bottom`
- Modal bottom sheets: add `contentInsetAdjustmentBehavior` or bottom safe area padding
- Push notifications: APNs key must be uploaded to EAS credentials

## Acceptance criteria
- [ ] `eas build --platform ios --profile farmer-ios-preview` completes without errors
- [ ] TestFlight build installs on iPhone
- [ ] No safe-area overflow (content not cut off by notch or home indicator)
- [ ] Push notifications arrive on iOS device (APNs path)
- [ ] All tap targets ≥ 44×44pt (iOS HIG minimum)

## Smoke test
1. Install TestFlight build on iPhone
2. Complete full onboarding flow — no safe area issues
3. Navigate all tabs — no content hidden behind notch
4. Receive a test push → tap → confirm navigation
5. Submit a harvest record form — keyboard dismissal works correctly
