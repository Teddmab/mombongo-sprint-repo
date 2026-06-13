# S0-01 — Repo Setup & Tooling

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S0-01 |
| Sprint | Sprint 0 — Infrastructure |
| Branch | `feature/s0-01-repo-setup` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 1.5 hours |
| Repo | `mombongo-web` (mombongo-grow-connect) |
| Status | ⚠️ PARTIALLY DONE — scaffold exists from Lovable, tooling needs adding |

---

## Context
Lovable already created the Vite + React + TypeScript + Tailwind scaffold.
`bun.lockb`, `components.json`, `eslint.config.js`, `vitest.config.ts` all exist.
**Do NOT recreate the scaffold.** This story adds the missing pieces only.

### What this builds
- Split Vitest config (unit vs integration)
- Playwright config (E2E + smoke)
- Firebase test setup files (mocks for unit, emulator for integration)
- Updated `package.json` scripts
- `.env.example`
- `e2e/` folder structure

---

## Step 1 — Remove Supabase (if present)

### Terminal
```bash
cd ~/mombongo-platform/mombongo-web

# Check if Supabase is installed
cat package.json | grep -i supabase

# If it appears:
bun remove @supabase/supabase-js
rm -rf src/integrations/supabase/
```

### Install Firebase + testing deps
```bash
bun add firebase@10
bun add -d \
  @firebase/rules-unit-testing \
  @testing-library/react \
  @testing-library/user-event \
  @testing-library/jest-dom \
  @vitest/coverage-v8 \
  @playwright/test \
  wait-on \
  react-i18next \
  i18next \
  i18next-browser-languagedetector \
  @tanstack/react-query \
  framer-motion \
  recharts \
  vite-plugin-pwa
```

### Regression
```bash
bun run build
# Must exit 0
```

---

## Step 2 — Split Vitest Config

### Lovable Prompt
```
The project already has vitest.config.ts. Do not delete it.
Create two new files alongside it:

vitest.unit.config.ts:
import { defineConfig, mergeConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  test: {
    name: 'unit',
    environment: 'jsdom',
    globals: true,
    setupFiles: ['src/test/setup.unit.ts'],
    include: ['src/**/__tests__/**/*.test.ts', 'src/**/__tests__/**/*.test.tsx'],
    exclude: ['src/**/__tests__/**/*.integration.test.ts'],
    coverage: { provider: 'v8', reporter: ['text', 'html'] },
  },
})

vitest.integration.config.ts:
import { defineConfig, mergeConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
  test: {
    name: 'integration',
    environment: 'node',
    globals: true,
    setupFiles: ['src/test/setup.integration.ts'],
    include: ['src/**/__tests__/**/*.integration.test.ts'],
    testTimeout: 30000,
  },
})

Create src/test/setup.unit.ts:
import '@testing-library/jest-dom'
import { vi } from 'vitest'

vi.mock('firebase/app', () => ({
  initializeApp: vi.fn(() => ({})),
  getApps: vi.fn(() => []),
  getApp: vi.fn(() => ({})),
}))
vi.mock('firebase/auth', () => ({
  getAuth: vi.fn(() => ({})),
  onAuthStateChanged: vi.fn(),
  signInWithEmailAndPassword: vi.fn(),
  createUserWithEmailAndPassword: vi.fn(),
  signInWithPopup: vi.fn(),
  GoogleAuthProvider: vi.fn(() => ({})),
  sendPasswordResetEmail: vi.fn(),
  updateProfile: vi.fn(),
  signOut: vi.fn(),
}))
vi.mock('firebase/firestore', () => ({
  getFirestore: vi.fn(() => ({})),
  enableIndexedDbPersistence: vi.fn().mockResolvedValue(undefined),
  collection: vi.fn(),
  doc: vi.fn(),
  getDoc: vi.fn(),
  getDocs: vi.fn(),
  setDoc: vi.fn(),
  updateDoc: vi.fn(),
  addDoc: vi.fn(),
  query: vi.fn(),
  where: vi.fn(),
  orderBy: vi.fn(),
  limit: vi.fn(),
  onSnapshot: vi.fn(),
  runTransaction: vi.fn(),
  serverTimestamp: vi.fn(() => new Date()),
  Timestamp: { fromMillis: vi.fn(), now: vi.fn() },
  increment: vi.fn(v => v),
  arrayUnion: vi.fn(),
}))
vi.mock('firebase/storage', () => ({
  getStorage: vi.fn(() => ({})),
  ref: vi.fn(),
  uploadBytes: vi.fn(),
  getDownloadURL: vi.fn(),
}))
vi.mock('firebase/functions', () => ({
  getFunctions: vi.fn(() => ({})),
  httpsCallable: vi.fn(() => vi.fn()),
}))
vi.mock('firebase/messaging', () => ({
  getMessaging: vi.fn(() => ({})),
  getToken: vi.fn(),
  onMessage: vi.fn(),
}))
vi.mock('@/lib/firebase', () => ({
  app: {}, db: {}, auth: {}, storage: {}, functions: {},
  isDevMode: vi.fn(() => false),
}))

Create src/test/setup.integration.ts:
export const clearFirestoreEmulator = async () => {
  const res = await fetch(
    'http://localhost:8080/emulator/v1/projects/mombongo-dev/databases/(default)/documents',
    { method: 'DELETE' }
  )
  if (!res.ok) console.warn('Could not clear Firestore emulator:', res.status)
}

export const createTestApp = (name: string) => {
  const { initializeApp, deleteApp } = require('firebase/app')
  const { getFirestore, connectFirestoreEmulator } = require('firebase/firestore')
  const { getAuth, connectAuthEmulator } = require('firebase/auth')

  const app = initializeApp(
    { projectId: 'mombongo-dev', apiKey: 'test-key' },
    `${name}-${Date.now()}`
  )
  const db = getFirestore(app)
  const auth = getAuth(app)
  connectFirestoreEmulator(db, 'localhost', 8080)
  connectAuthEmulator(auth, 'http://localhost:9099', { disableWarnings: true })
  return { app, db, auth, cleanup: () => deleteApp(app) }
}

Update package.json scripts (add these alongside existing scripts):
"test:unit": "vitest run --config vitest.unit.config.ts",
"test:unit:watch": "vitest --config vitest.unit.config.ts",
"test:integration": "vitest run --config vitest.integration.config.ts",
"test:e2e": "playwright test",
"test:smoke": "playwright test --config playwright.smoke.config.ts",
"test:ci": "bun run test:unit && bun run test:integration",
"emulator:start": "firebase emulators:start --only auth,firestore,storage,functions --import ./emulator-seed --export-on-exit ./emulator-seed"
```

### Regression
```bash
bun run test:unit
# Expected: passes (0 tests is fine at this stage)
bun run typecheck
# Expected: exits 0
```

---

## Step 3 — Playwright Config

### Lovable Prompt
```
Create playwright.config.ts:
import { defineConfig, devices } from '@playwright/test'
export default defineConfig({
  testDir: './e2e',
  fullyParallel: false,
  retries: process.env.CI ? 1 : 0,
  workers: 1,
  reporter: [['html', { open: 'never' }], ['list']],
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL ?? 'http://localhost:8080',
    trace: 'on-first-retry',
  },
  projects: [{ name: 'mobile-chrome', use: { ...devices['Pixel 7'] } }],
})

Create playwright.smoke.config.ts:
import { defineConfig, devices } from '@playwright/test'
export default defineConfig({
  testDir: './e2e/smoke',
  retries: 2,
  use: { baseURL: process.env.PLAYWRIGHT_BASE_URL ?? 'https://app.mombongo.coop' },
  projects: [{ name: 'chromium', use: { ...devices['Pixel 7'] } }],
})

Create directory structure:
e2e/
  helpers/
    auth.helper.ts
    seed.helper.ts
  smoke/
    homepage.spec.ts

e2e/helpers/auth.helper.ts:
import { Page } from '@playwright/test'
export const authHelper = {
  async loginAs(page: Page, role: 'investor' | 'farmer' | 'agent' | 'admin' | 'merchant') {
    const emails: Record<string, string> = {
      investor: process.env.TEST_INVESTOR_EMAIL ?? 'investor@test.com',
      farmer: 'farmer@test.com',
      agent: 'agent@test.com',
      admin: process.env.TEST_ADMIN_EMAIL ?? 'admin@test.com',
      merchant: 'merchant@test.com',
    }
    const password = process.env.TEST_INVESTOR_PASSWORD ?? 'Mombongo2026!'
    await page.goto('/auth')
    await page.fill('[placeholder*="email" i]', emails[role])
    await page.fill('[type="password"]', password)
    await page.click('[type="submit"]')
    await page.waitForURL(/home/, { timeout: 10000 })
  }
}

e2e/smoke/homepage.spec.ts:
import { test, expect } from '@playwright/test'
test('app loads without JS errors', async ({ page }) => {
  const errors: string[] = []
  page.on('pageerror', err => errors.push(err.message))
  await page.goto('/')
  await page.waitForLoadState('networkidle')
  const firebaseErrors = errors.filter(e => e.includes('Firebase') || e.includes('firestore'))
  expect(firebaseErrors).toHaveLength(0)
})
test('page has Mombongo in title', async ({ page }) => {
  await page.goto('/')
  await expect(page).toHaveTitle(/[Mm]ombongo/)
})
```

### Terminal
```bash
bunx playwright install --with-deps chromium
```

### Regression
```bash
bun run typecheck && bun run lint && bun run build
# All must exit 0
```

---

## Step 4 — .env.example

### Terminal
```bash
cat > .env.example << 'EOF'
VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=
VITE_FIREBASE_VAPID_KEY=
VITE_ENVIRONMENT=development
VITE_DEV_MODE=true
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
VITE_PAWAPAY_ENV=sandbox
EOF

grep ".env.local" .gitignore || echo ".env.local" >> .gitignore
```

---

## ✅ Milestone — S0-01 Complete

```bash
bun run typecheck   # ✅ exits 0
bun run lint        # ✅ exits 0 (1 warning, 0 errors)
bun run test:unit   # ✅ passes (passWithNoTests)
bun run build       # ✅ exits 0
```

## 🏁 PR Checklist
- [x] `vitest.unit.config.ts` and `vitest.integration.config.ts` created
- [x] `src/test/setup.unit.ts` — all Firebase modules mocked
- [x] `src/test/setup.integration.ts` — emulator helpers (dynamic imports for ESM)
- [x] `playwright.config.ts` — Pixel 7 profile
- [x] `e2e/helpers/auth.helper.ts` created
- [x] All `test:*` scripts in `package.json`
- [x] `.env.example` committed (no values)
- [x] `.env.local` in `.gitignore`
- [ ] `bun run test:ci` exits 0 — deferred: requires integration tests (S0-03+)

## 📋 Notes
- PR: https://github.com/Teddmab/mombongo-web/pull/1
- CI workflow added: `.github/workflows/ci.yml`
- `@typescript-eslint/no-explicit-any` turned off globally (Lovable scaffold uses `any` extensively)
- `src/components/ui/**` excluded from ESLint (auto-generated shadcn components)
- `vitest.unit.config.ts` has `passWithNoTests: true` so CI stays green until tests are written
