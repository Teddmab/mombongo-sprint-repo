# S7-03 — QA — End-to-End Smoke Tests (Playwright)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S7-03 |
| Sprint | Sprint 7 — PWA & Production |
| Branch | `feature/s7-03-e2e-tests` |
| Merges into | `dev` |
| Estimate | 2.5h |
| Dependencies | S7-02 (all features complete) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Playwright setup, 5 core user journey tests |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — Install Playwright

```bash
npm install -D @playwright/test
npx playwright install chromium
```

Create `playwright.config.ts`:

```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir:       './e2e',
  timeout:       30_000,
  retries:       1,
  reporter:      'html',
  use: {
    baseURL:     'http://localhost:5173',
    trace:       'on-first-retry',
    screenshot:  'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile',   use: { ...devices['Pixel 5'] } },
  ],
  webServer: {
    command:  'npm run dev',
    url:      'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
})
```

Add `"test:e2e": "playwright test"` and `"test:e2e:ui": "playwright test --ui"` to `package.json`.

### Step 2 — Test helpers

Create `e2e/helpers/auth.ts`:

```typescript
import { Page } from '@playwright/test'

// Use a dedicated E2E test account seeded in Firebase Emulator or staging
const E2E_EMAIL    = process.env.E2E_EMAIL    ?? 'e2e@mombongo.test'
const E2E_PASSWORD = process.env.E2E_PASSWORD ?? 'E2EPassword123!'

export async function loginAsTestUser(page: Page) {
  await page.goto('/login')
  await page.fill('[data-testid="email-input"]', E2E_EMAIL)
  await page.fill('[data-testid="password-input"]', E2E_PASSWORD)
  await page.click('[data-testid="signin-btn"]')
  await page.waitForURL('/')
}
```

### Step 3 — Core smoke tests

Create `e2e/auth.spec.ts`:
```typescript
import { test, expect } from '@playwright/test'
import { loginAsTestUser } from './helpers/auth'

test('user can log in and see home screen', async ({ page }) => {
  await loginAsTestUser(page)
  await expect(page.locator('[data-testid="home-screen"]')).toBeVisible()
})

test('user can log out', async ({ page }) => {
  await loginAsTestUser(page)
  await page.click('[data-testid="profile-menu"]')
  await page.click('[data-testid="logout-btn"]')
  await page.waitForURL('/login')
})
```

Create `e2e/market.spec.ts`:
```typescript
test('user can browse products and open detail', async ({ page }) => {
  await loginAsTestUser(page)
  await page.click('[data-testid="nav-market"]')
  await expect(page.locator('[data-testid="market-screen"]')).toBeVisible()
  // Click first product card
  await page.locator('[data-testid^="product-card-"]').first().click()
  await expect(page.locator('[data-testid="product-invest-cta"]')).toBeVisible()
})
```

Create `e2e/bourse.spec.ts`:
```typescript
test('bourse screen shows opportunities', async ({ page }) => {
  await loginAsTestUser(page)
  await page.click('[data-testid="nav-bourse"]')
  await expect(page.locator('[data-testid="bourse-screen"]')).toBeVisible()
  // At least one opportunity card visible
  await expect(page.locator('[data-testid^="opportunity-card-"]').first()).toBeVisible()
})

test('bourse detail opens from opportunity card', async ({ page }) => {
  await loginAsTestUser(page)
  await page.goto('/bourse')
  await page.locator('[data-testid^="opportunity-card-"]').first().click()
  await expect(page.locator('[data-testid="bourse-invest-cta"]')).toBeVisible()
})
```

Create `e2e/portfolio.spec.ts`:
```typescript
test('portfolio screen is reachable', async ({ page }) => {
  await loginAsTestUser(page)
  await page.goto('/portfolio')
  await expect(page.locator('[data-testid="portfolio-screen"]')).toBeVisible()
})
```

Create `e2e/academia.spec.ts`:
```typescript
test('academia screen shows course cards', async ({ page }) => {
  await loginAsTestUser(page)
  await page.click('[data-testid="nav-academia"]')
  await expect(page.locator('[data-testid="academia-screen"]')).toBeVisible()
})

test('can view course detail', async ({ page }) => {
  await loginAsTestUser(page)
  await page.goto('/academia')
  await page.locator('[data-testid^="course-card-"]').first().click()
  await expect(page.locator('[data-testid="course-enroll-btn"]')).toBeVisible()
})
```

### Step 4 — CI integration

Add to `.github/workflows/ci.yml`:

```yaml
e2e:
  runs-on: ubuntu-latest
  needs: build
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: { node-version: '20' }
    - run: npm ci
    - run: npx playwright install --with-deps chromium
    - run: npm run build
    - run: npm run test:e2e
      env:
        E2E_EMAIL:    ${{ secrets.E2E_EMAIL }}
        E2E_PASSWORD: ${{ secrets.E2E_PASSWORD }}
    - uses: actions/upload-artifact@v4
      if: failure()
      with:
        name: playwright-report
        path: playwright-report/
```

---

## ✅ Definition of Done
- [ ] Playwright installed and configured with chromium + mobile viewports
- [ ] 5 smoke test files pass against `npm run dev`
- [ ] All `data-testid` attributes referenced in tests are present in components
- [ ] CI workflow runs `test:e2e` on pull requests
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s7-03): Playwright e2e smoke tests — auth, market, bourse, portfolio, academia"
git push origin feature/s7-03-e2e-tests
```
