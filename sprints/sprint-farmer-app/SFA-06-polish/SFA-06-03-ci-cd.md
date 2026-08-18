# SFA-06-03 — CI/CD: GitHub Actions + EAS Builds (Both Apps)

## Context
Port of SM-00-00 and SM-00-01, adapted for the per-role EAS build architecture. Both the Farmer App and Agent App are built from the same `mombongo-mobile` codebase. CI must run lint + tests on every PR, and trigger EAS preview builds when branches merge to `dev`.

This story covers `mombongo-mobile` CI/CD only. The `mombongo-functions` and `mombongo-web` repos have their own CI (not in scope here).

## Scope
- Add ESLint + Prettier config to `mombongo-mobile` (if missing)
- Set up Jest with `jest-expo` preset + coverage threshold
- Create `.github/workflows/ci.yml` — runs on all PRs: lint + type-check + Jest
- Create `.github/workflows/eas-preview.yml` — triggers on `dev` branch push; builds both `farmer-preview` and `agent-preview` profiles
- Create `.github/workflows/eas-production.yml` — manual trigger only; builds `farmer` and `agent` production profiles

## GitHub Actions secrets required (add in GitHub repo settings)
- `EXPO_TOKEN` — Expo access token (for EAS CLI)
- `GOOGLE_SERVICES_JSON` — base64-encoded `google-services.json` (Android Firebase config)

## Files to create
- `.github/workflows/ci.yml`
- `.github/workflows/eas-preview.yml`
- `.github/workflows/eas-production.yml`
- `jest.config.js` (if missing)
- `.eslintrc.js` (if missing)

## Implementation

### `.github/workflows/ci.yml`
```yaml
name: CI

on:
  pull_request:
    branches: [dev, main]

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - working-directory: mombongo-mobile
        run: npm ci
      - working-directory: mombongo-mobile
        run: npm run lint
      - working-directory: mombongo-mobile
        run: npx tsc --noEmit
      - working-directory: mombongo-mobile
        run: npx jest --coverage --coverageThreshold='{"global":{"lines":60}}'
```

### `.github/workflows/eas-preview.yml`
```yaml
name: EAS Preview Build

on:
  push:
    branches: [dev]
    paths: ['mombongo-mobile/**']

jobs:
  build-farmer-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - uses: expo/expo-github-action@v8
        with: { eas-version: latest, token: '${{ secrets.EXPO_TOKEN }}' }
      - working-directory: mombongo-mobile
        run: npm ci
      - working-directory: mombongo-mobile
        run: echo '${{ secrets.GOOGLE_SERVICES_JSON }}' | base64 -d > google-services.json
      - working-directory: mombongo-mobile
        run: eas build --platform android --profile farmer-preview --non-interactive

  build-agent-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - uses: expo/expo-github-action@v8
        with: { eas-version: latest, token: '${{ secrets.EXPO_TOKEN }}' }
      - working-directory: mombongo-mobile
        run: npm ci
      - working-directory: mombongo-mobile
        run: echo '${{ secrets.GOOGLE_SERVICES_JSON }}' | base64 -d > google-services.json
      - working-directory: mombongo-mobile
        run: eas build --platform android --profile agent-preview --non-interactive
```

### `.github/workflows/eas-production.yml`
```yaml
name: EAS Production Build

on:
  workflow_dispatch:
    inputs:
      app:
        description: 'Which app to build'
        required: true
        type: choice
        options: [farmer, agent, both]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: expo/expo-github-action@v8
        with: { eas-version: latest, token: '${{ secrets.EXPO_TOKEN }}' }
      - working-directory: mombongo-mobile
        run: npm ci
      - if: inputs.app == 'farmer' || inputs.app == 'both'
        working-directory: mombongo-mobile
        run: eas build --platform android --profile farmer --non-interactive
      - if: inputs.app == 'agent' || inputs.app == 'both'
        working-directory: mombongo-mobile
        run: eas build --platform android --profile agent --non-interactive
```

### `jest.config.js`
```javascript
module.exports = {
  preset: 'jest-expo',
  setupFilesAfterFramework: ['<rootDir>/src/test/setup.ts'],
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg))',
  ],
  coverageThreshold: {
    global: { lines: 60, branches: 50 },
  },
}
```

## ⚠️ Manual steps required (Teddy)
1. Add `EXPO_TOKEN` secret to `mombongo-mobile` GitHub repo (Settings → Secrets → Actions)
2. Add `GOOGLE_SERVICES_JSON` secret (base64-encoded content of `google-services.json`)
3. Run first EAS build manually to verify `eas.json` profiles are valid before relying on CI

## Acceptance criteria
- [ ] CI runs on every PR to `dev` or `main`
- [ ] Lint failures block merge (PR shows failing check)
- [ ] Jest coverage < 60% blocks merge
- [ ] Push to `dev` triggers farmer-preview + agent-preview EAS builds automatically
- [ ] Production builds require manual dispatch (no accidental production releases)

## Smoke test
1. Open a test PR with a lint error → confirm CI fails with lint error
2. Fix lint, push → CI passes
3. Merge to `dev` → confirm two EAS builds start in Expo dashboard (farmer-preview + agent-preview)
4. Trigger production workflow manually with `app: farmer` → confirm only farmer build starts
