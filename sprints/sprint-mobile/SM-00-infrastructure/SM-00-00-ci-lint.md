# SM-00-00 — Add ESLint to CI

**Sprint:** SM-00 · Infrastructure  
**Branch:** `feature/sm-00-infrastructure`

## Context
The mobile CI (`ci.yml`) only runs `npx tsc --noEmit`. The web and admin apps also run ESLint as a separate step. Adding lint to mobile CI catches formatting and import errors before they reach the EAS build queue.

## Acceptance criteria
- [ ] `package.json` has `"lint": "eslint . --ext .ts,.tsx --max-warnings 0"` script
- [ ] `.eslintrc.js` (or flat config) is present with expo + react-hooks + import rules
- [ ] `ci.yml` has a `lint` job that runs on PR and push to dev
- [ ] `ci.yml` jobs run in parallel (lint does not depend on typecheck)
- [ ] All existing files pass lint with 0 warnings

## Implementation notes
- Base config: `eslint-config-expo` (already installed with Expo SDK)
- Add `eslint-plugin-react-hooks` if not already a transitive dep
- `.eslintignore`: `node_modules/`, `dist/`, `.expo/`
- CI step: `npm run lint`
