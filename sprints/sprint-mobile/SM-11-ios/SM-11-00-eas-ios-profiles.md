# SM-11-00 — iOS EAS build profiles

**Sprint:** SM-11 · iOS  
**Branch:** `feature/sm-11-ios`

## Context
`eas.json` only has Android build configuration. iOS support is missing entirely. This story adds iOS profiles and the necessary provisioning setup.

## Acceptance criteria
- [ ] `eas.json` `development` profile: `"ios": { "simulator": true }` for local testing
- [ ] `eas.json` `preview` profile: `"ios": { "distribution": "internal" }` for TestFlight internal
- [ ] `eas.json` `production` profile: `"ios": { "distribution": "store" }` for App Store
- [ ] `app.json` has `expo.ios.bundleIdentifier: "com.mombongo.app"`
- [ ] `app.json` has `expo.ios.supportsTablet: false` (phone-only for MVP)
- [ ] CI `preview.yml` updated to also build iOS (or separate `preview-ios.yml` triggered manually)
- [ ] Production `production.yml` updated to include `--platform ios` option (manual dispatch only for MVP)
- [ ] EAS credentials auto-provisioned: `eas credentials` sets up Apple distribution certificate + provisioning profile

## Pre-requisites (must do manually)
- Apple Developer Account membership (paid $99/yr)
- App Store Connect app created: "Mombongo" with bundle ID `com.mombongo.app`
- EAS CLI: `eas credentials --platform ios` to generate/link credentials

## Implementation notes
- `google-services.json` is for Android only; iOS needs `GoogleService-Info.plist` from Firebase Console → add to `app.json` `expo.ios.googleServicesFile`
- Google Sign-In on iOS: `REVERSED_CLIENT_ID` must be added to `app.json` URL schemes
- `expo-notifications` on iOS requires push notifications capability and APNs key in EAS credentials
