# SM-00-01 — Jest + unit test setup

**Sprint:** SM-00 · Infrastructure  
**Branch:** `feature/sm-00-infrastructure`

## Context
No test framework is configured in the mobile app. The web app uses Vitest with react-testing-library. For React Native we use Jest + `jest-expo` preset + `@testing-library/react-native`.

## Acceptance criteria
- [ ] `jest.config.js` (or package.json `jest` field) uses `jest-expo` preset
- [ ] `src/test/setup.ts` mocks: `firebase`, `expo-router`, `react-native-safe-area-context`, `@react-native-async-storage/async-storage`
- [ ] At least 3 unit tests passing: `usePortfolioStats`, `formatUsd`, one hook guard test
- [ ] `package.json` has `"test": "jest"` and `"test:ci": "jest --ci --coverage"`
- [ ] `ci.yml` has a `test` job running `npm run test:ci`

## Implementation notes
- Mocking firebase: return `httpsCallable` that resolves with mock data
- Hook tests: wrap in `QueryClientWrapper` to provide QueryClient
- Do not test Expo-specific native modules (camera, notifications) — mock them
- Coverage threshold: 60% for `src/hooks/**`
