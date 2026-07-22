# S0-02-SPECIAL — Decouple Admin from mombongo-web

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S0-02-SPECIAL |
| Sprint | Sprint 0 — Infrastructure |
| Branch (web) | `feature/s0-02s-remove-admin` |
| Branch (admin) | `feature/s0-02s-init-admin` |
| Owner | Afrotouch OU |
| Estimate | 2 hours |
| Must complete before | S0-02 (Firebase setup) |
| Priority | P0 — Do this FIRST before any feature work |

---

## Context

### Problem
Lovable generated the investor-facing UI **and** the admin dashboard inside the
same repo (`mombongo-grow-connect` / `mombongo-web`). This creates two problems:

1. **Security** — admin code ships to investors. Any investor who inspects the
   JS bundle can see admin routes, KPI logic, and user management code.
2. **Deployment** — you cannot deploy admin independently. A bug in admin blocks
   investor app deploys and vice versa.

### Solution
- **`mombongo-web`** — investor PWA only. No admin code whatsoever.
- **`mombongo-admin`** — standalone Vite + React app. Separate Cloudflare Pages
  project. Separate Firebase web app registration. Only accessible by djuna sambil
  and Patrick. Deployed to `admin.mombongo.coop`.

### What moves
```
STAYS in mombongo-web:               MOVES to mombongo-admin:
─────────────────────────            ──────────────────────────────
src/pages/HomeScreen                 src/pages/admin/AdminDashboard
src/pages/MarketScreen               src/pages/admin/AdminUsers
src/pages/BourseScreen               src/pages/admin/AdminTransactions
src/pages/FinancingScreen            src/pages/admin/AdminFinancing
src/pages/AcademiaScreen             src/pages/admin/AdminBourse
src/pages/ProfileScreen              src/components/admin/* (any admin components)
src/pages/PortfolioScreen            src/services/admin.service.ts
src/pages/NotificationsScreen        Any route starting with /admin/*
src/components/layout/*              AdminRoute guard
src/services/* (non-admin)
```

---

## Part A — Clean mombongo-web

**Repo:** `mombongo-web` | **Branch:** `feature/s0-02s-remove-admin`

```bash
cd ~/mombongo-platform/mombongo-web
git checkout -b feature/s0-02s-remove-admin
```

### Step A-1 — Audit what Lovable put in admin

Open VS Code → `mombongo-web/src` and note every file related to admin:

```bash
# In terminal — find all admin files
find src -type f | grep -i admin
find src -type f | grep -i "admin\|AdminDashboard\|AdminUsers\|AdminRoute"
```

List every file you find — these all move to mombongo-admin.

---

### Step A-2 — Remove admin from mombongo-web

### Lovable Prompt (paste into mombongo-web Lovable project)
```
Remove ALL admin-related code from this project. This code is moving to a
separate dedicated repository.

Delete these files entirely:
- src/pages/admin/ (entire folder and all files inside)
- src/components/admin/ (entire folder if it exists)
- src/services/admin.service.ts (if it exists)
- src/components/layout/AdminRoute.tsx

Update src/AppRoutes.tsx:
- Remove ALL routes that start with /admin
- Remove the AdminRoute import
- Remove the AdminDashboard, AdminUsers, AdminTransactions,
  AdminFinancing, AdminBourse imports
- Keep all other routes exactly as they are

Update src/store/AuthContext.tsx or wherever useRole is defined:
- Keep the isAdmin boolean in useRole() — the web app still needs
  to know if a user is admin (e.g. to show an "Admin panel" link)
- But remove any admin-specific logic or redirects

Check src/main.tsx and remove any admin provider or import if present.

After removing, make sure the app still compiles. The /admin routes
simply won't exist anymore — that is correct and expected.
Do not create placeholder stubs for admin routes.
```

### Regression
```bash
bun run typecheck
# Must exit 0 — no broken imports referencing deleted admin files

bun run build
# Must exit 0 — dist/ must not contain any admin page code

# Verify: admin code is gone from the bundle
grep -r "AdminDashboard\|AdminUsers\|admin-dashboard" dist/ || echo "✅ No admin code in bundle"
```

---

### Step A-3 — Add admin link to investor app (optional but clean)

### Lovable Prompt
```
In src/pages/ProfileScreen.tsx, add a small "Admin Panel" button that is
only visible when useRole().isAdmin is true.

The button should:
- Show only when the logged-in user has role === 'admin'
- Display: "⚙️ Admin Panel →"
- On press: window.open('https://admin.mombongo.coop', '_blank')
  (opens the separate admin app in a new tab)
- Style: outlined button, small, at the bottom of the profile screen

This is the only connection between the investor app and the admin app.
```

### Regression
```bash
bun run test:unit && bun run build
```

### Commit A
```bash
git add -A
git commit -m "feat(s0-02s): remove admin code from investor app, add admin link on profile"
git push origin feature/s0-02s-remove-admin
# Open PR → dev
```

---

## Part B — Initialize mombongo-admin

**Repo:** `mombongo-admin` | **Branch:** `feature/s0-02s-init-admin`

```bash
cd ~/mombongo-platform/mombongo-admin
git checkout -b feature/s0-02s-init-admin
```

### Step B-1 — Create a new Lovable project for admin

1. Go to **lovable.dev** → New project
2. Name: `mombongo-admin`
3. Connect to GitHub repo: `Teddmab/mombongo-admin`
4. Let Lovable scaffold the base project

Then paste this prompt:

### Lovable Prompt (paste into the NEW mombongo-admin Lovable project)
```
Create a clean admin dashboard application. This is a DESKTOP-FIRST app
(not mobile). It will be used by djuna sambil and Patrick to manage the
Mombongo cooperative platform.

Install these packages:
  firebase@10
  @tanstack/react-query@5
  react-router-dom@6
  react-i18next i18next
  recharts
  lucide-react
  framer-motion
  @testing-library/react @testing-library/user-event @testing-library/jest-dom
  vite-plugin-pwa

Project structure to create:
src/
  lib/
    firebase.ts       ← same Firebase config as mombongo-web (reads VITE_* env vars)
    utils.ts          ← formatUsd, formatCdf helpers
  types/
    index.ts          ← same TypeScript interfaces as mombongo-web (copy exactly)
  store/
    AuthContext.tsx   ← same AuthProvider pattern as mombongo-web
  hooks/
    useAuth.ts
    useRole.ts        ← same as mombongo-web
  services/
    admin.service.ts  ← AdminService class (KPIs, user management, reports)
  pages/
    LoginScreen.tsx          ← simple email/password login (no role selector)
    AdminDashboard.tsx       ← stub
    AdminUsers.tsx           ← stub
    AdminTransactions.tsx    ← stub
    AdminFinancing.tsx       ← stub
    AdminBourse.tsx          ← stub
  components/
    layout/
      AdminShell.tsx    ← sidebar (240px) + main content area, desktop only
      AdminSidebar.tsx  ← navigation links
      AdminHeader.tsx   ← top bar with user name + logout

Layout: AdminShell wraps all protected pages.
  Sidebar (240px, fixed left):
    🌿 Mombongo Admin (logo + subtitle)
    Navigation links (NavLink from react-router-dom):
      📊 Dashboard → /dashboard
      👥 Utilisateurs → /users
      💰 Transactions → /transactions
      🌾 Financement → /financing
      🏛️ Bourse → /bourse
    Bottom: logged-in user name + role + "Déconnexion" button
  Main content: flex-1, overflow-y-auto, bg-gray-50, p-8

Routes (all protected — redirect to /login if not authenticated):
  / → redirect to /dashboard
  /login → LoginScreen (no sidebar)
  /dashboard → AdminDashboard
  /users → AdminUsers
  /transactions → AdminTransactions
  /financing → AdminFinancing
  /bourse → AdminBourse

Auth guard: if logged in but role !== 'admin': show
  "Accès refusé. Ce panneau est réservé aux administrateurs."
  + logout button

Color scheme: professional, not the green investor theme.
  Primary: #1E3A5F (dark navy)
  Accent: #F4A11B (amber — same as Mombongo brand)
  Background: #F8FAFC (light gray)
  Sidebar bg: #1E3A5F (dark navy), text white

Title in index.html: "Mombongo — Admin"
```

### Step B-2 — Add vitest configs and test setup

### Lovable Prompt
```
Add the same test configuration as mombongo-web:

Create vitest.unit.config.ts and vitest.integration.config.ts
(same content as mombongo-web — unit uses jsdom + Firebase mocks,
integration uses node + Firebase Emulator).

Create src/test/setup.unit.ts with all Firebase modules mocked
(identical to mombongo-web/src/test/setup.unit.ts).

Add to package.json scripts:
"test:unit": "vitest run --config vitest.unit.config.ts",
"test:unit:watch": "vitest --config vitest.unit.config.ts",
"test:integration": "vitest run --config vitest.integration.config.ts",
"test:ci": "bun run test:unit",
"typecheck": "tsc --noEmit",
"lint": "eslint . --ext ts,tsx"
```

### Step B-3 — Copy workflow files

```bash
# Copy CI/CD workflows from sprint-repo
mkdir -p .github/workflows
cp ~/mombongo-platform/mombongo-sprint-repo/.github/workflows/ci.yml .github/workflows/
cp ~/mombongo-platform/mombongo-sprint-repo/.github/workflows/staging.yml .github/workflows/
cp ~/mombongo-platform/mombongo-sprint-repo/.github/workflows/production.yml .github/workflows/
```

Then update the workflow files — change ONE thing in staging.yml and production.yml:

```yaml
# staging.yml — change projectName:
projectName: mombongo-admin   # was mombongo-web

# production.yml — change projectName and PRODUCTION_URL:
projectName: mombongo-admin
PRODUCTION_URL: https://admin.mombongo.coop
```

### Step B-4 — Create .env.local

```bash
# Same Firebase keys as mombongo-web (same mombongo-dev project)
# Admin uses the SAME Firebase backend — just a different frontend
touch .env.local
```

Add to `.env.local`:
```
VITE_FIREBASE_API_KEY=          ← same as mombongo-web
VITE_FIREBASE_AUTH_DOMAIN=mombongo-dev.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=mombongo-dev
VITE_FIREBASE_STORAGE_BUCKET=mombongo-dev.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=   ← same as mombongo-web
VITE_FIREBASE_APP_ID=           ← same as mombongo-web
VITE_ENVIRONMENT=development
VITE_DEV_MODE=true
```

> ⚠️ Same Firebase project, same keys. The Security Rules already restrict
> admin-only collections to role === 'admin'. No need for a separate Firebase project.

### Regression
```bash
bun install
bun run typecheck   # ✅ exits 0
bun run build       # ✅ exits 0
```

### Commit B
```bash
git add -A
git commit -m "feat(s0-02s): initialize mombongo-admin with sidebar layout, auth guard, stub pages"
git push origin feature/s0-02s-init-admin
# Open PR → dev (in mombongo-admin repo)
```

---

## Part C — Cloudflare Pages for mombongo-admin

```
1. dash.cloudflare.com → Pages → Create a project
2. Connect: Teddmab/mombongo-admin
3. Project name: mombongo-admin      ← must match workflow files
4. Framework: Vite
5. Build command: bun run build
6. Output directory: dist
7. Add same VITE_* environment variables as mombongo-web
8. Deploy

Custom domain (when ready): admin.mombongo.coop
```

Then add GitHub secrets to `mombongo-admin` repo:
```
CLOUDFLARE_API_TOKEN    ← same token as mombongo-web
CLOUDFLARE_ACCOUNT_ID   ← same account ID
STAGING_FIREBASE_*      ← same Firebase keys
FIREBASE_TOKEN          ← same CI token
TEST_ADMIN_EMAIL        ← admin@test.com
TEST_ADMIN_PASSWORD     ← Mombongo2026!
```

---

## ✅ Definition of Done

**mombongo-web:**
- [ ] Zero files matching `*admin*` in `src/` (except the "Admin Panel" link on ProfileScreen)
- [ ] `/admin` route does not exist
- [ ] `bun run build` — `dist/` contains no admin page code
- [ ] `bun run test:ci` exits 0

**mombongo-admin:**
- [ ] Standalone Vite app running at `localhost:8080`
- [ ] Login → redirects non-admin users with "Accès refusé"
- [ ] Login as `admin@test.com` → reaches `/dashboard`
- [ ] Sidebar shows all 5 navigation links
- [ ] `bun run typecheck` exits 0
- [ ] `bun run build` exits 0
- [ ] Deployed to Cloudflare Pages

---

## Architecture after this sprint

```
mombongo-web (app.mombongo.coop)
  └── Investor PWA — no admin code

mombongo-admin (admin.mombongo.coop)
  └── Admin dashboard — only accessible with role: admin
  └── Same Firebase backend (mombongo-dev / mombongo-prod)
  └── Security Rules enforce role === 'admin' at database level

Both apps → same Firestore, same Auth, same Storage
```

---

## PR Description Template

### mombongo-web PR
```
## feat(s0-02s): decouple admin — remove from investor app

### Changes
- Deleted src/pages/admin/ (entire folder)
- Deleted src/services/admin.service.ts
- Deleted src/components/layout/AdminRoute.tsx
- Removed /admin/* routes from AppRoutes.tsx
- Added "Admin Panel →" link on ProfileScreen (admin role only, opens admin.mombongo.coop)

### Security
- Admin code no longer ships in the investor bundle
- Admin routes no longer accessible from investor app URL
```

### mombongo-admin PR
```
## feat(s0-02s): initialize mombongo-admin standalone app

### Changes
- New Vite + React + TypeScript project
- Desktop-first AdminShell (240px sidebar + main content)
- Auth guard: role !== admin → access denied screen
- 5 stub pages: Dashboard, Users, Transactions, Financing, Bourse
- Same Firebase config as mombongo-web (shared backend)
- CI/CD workflows (Cloudflare Pages, mombongo-admin project)
```