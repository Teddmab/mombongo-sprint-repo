# SP-03 — Test Fixes, Sprint Doc Maintenance & S2-01 Market Wiring

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-03 |
| Sprint | Patch — Point-in-time corrections & quick implementations |
| Branch | `feature/sp-03-fixes-and-polish` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Date | 2026-06-03 |
| Type | Fixes + first data-wiring sprint |

## Scope — `mombongo-web` + `mombongo-sprint-repo`

Four areas:
1. Fix 2 broken routing unit tests introduced by SP-02 changes to `AppContext`
2. Sprint doc maintenance — update 3 stale docs, rename 7 mismatched files, establish DONE convention
3. Implement S2-01 — wire `/market` investor view to `useProducts` (first screen off mock data)
4. Replace all CinetPay references with **PawaPay** (mobile money) + **Stripe** (Visa/Mastercard)

---

## Changes

### 1. Test setup fix — `src/test/setup.ts`

**Problem:** SP-02 modified `AppContext.tsx` to persist `lang` and `role` to `localStorage` during state initialization. The routing test (`app-routing.test.tsx`) renders the full `<App />` tree but runs under `vitest.config.ts` which uses `setup.ts` — a minimal setup with no `localStorage` mock and no `react-i18next` mock.

Two tests broke:
- `redirects root route to language selection` → `TypeError: localStorage.getItem is not a function`
- `renders the in-app not found page for unknown routes` → `TypeError: i18n.changeLanguage is not a function`

**Fix:** Added to `src/test/setup.ts`:
- A localStorage mock (plain object, not vi.fn — no vitest dependency in the common setup)
- `vi.mock("react-i18next", ...)` with `useTranslation` returning `{ t, i18n: { changeLanguage: vi.fn() } }`

**Result:** 49/49 tests passing, 0 TypeScript errors.

---

### 2. Sprint doc maintenance — `mombongo-sprint-repo`

#### 2a. Stale sprint docs updated

| File | What changed |
|------|-------------|
| `sprint-4-financing/S4-03-agent-report.md` | Added "Status: UI complete via SP-02" section. SP-02 built the full 601-line role-dispatched form. Remaining work scoped to: wire farmer selector + submit handler to Firestore/Functions. |
| `sprint-6-payments-and-notifications/S6-01-payments.md` | Added "Status: Wallet UI complete via SP-02." Completely rewritten: CinetPay removed, replaced with **PawaPay** (mobile money — `initiateDeposit`, `pawapayWebhook`, `initiateWithdraw`) + **Stripe** (card — `createStripePaymentIntent`, `stripeWebhook`). |
| `sprint-2-marketplace/S2-01-product-list.md` | Added "Current State" clarification: `useProducts.ts` exists from S1-03; `MarketScreen.tsx` was extended with role views in SP-01; investor view (`DesktopMarket` / `MobileMarket`) still imports from mock. Scope now explicitly targets investor view only. |

#### 2b. Mismatched file names corrected

Sprint 6 — file names previously reflected a discarded admin-screen plan:

| Old name | New name | Content |
|----------|----------|---------|
| `S6-01-admin-dashboard.md` | `S6-01-payments.md` | PawaPay Mobile Money + Stripe Card Flow |
| `S6-02-admin-users.md` | `S6-02-fcm-notifications.md` | FCM Push Notifications |
| `S6-03-admin-financing.md` | `S6-03-admin-users.md` | Admin User Management |
| `S6-04-admin-bourse.md` | `S6-04-admin-kpi-dashboard.md` | Admin Enhanced KPI Dashboard |

Sprint 7:

| Old name | New name | Content |
|----------|----------|---------|
| `S7-02-fcm-notifications.md` | `S7-02-polish.md` | i18n Completeness, Animations & Accessibility |
| `S7-03-cinetpay.md` | `S7-03-e2e-tests.md` | End-to-End Smoke Tests (Playwright) |
| `S7-04-polish.md` | `S7-04-production-deploy.md` | Production Deploy Checklist & Final Config |

#### 2c. DONE convention established

Sprint docs are renamed with a `-DONE` suffix on the filename when the story is fully implemented and tests pass.

Example: `S2-01-product-list.md` → `S2-01-product-list-DONE.md`

This convention is documented in `CLAUDE.md` at the project root.

---

### 3. S2-01 — Wire MarketScreen investor view to useProducts

**Problem:** `DesktopMarket` and `MobileMarket` in `MarketScreen.tsx` still import `products` directly from `src/data/mock.ts` despite `useProducts.ts` existing since S1-03.

**Fix:**
- Removed `import { products, Category } from "@/data/mock"` from `MarketScreen.tsx`
- Added `import { useProducts } from "@/hooks/useProducts"` with `const { data: products = [], isLoading, isError } = useProducts()` inside both sub-components
- Added SkeletonLoader grid while loading (8 cards)
- Added empty state (Sprout icon + message) when array is empty
- Added error state when query fails
- Fixed sidebar category counts to use the reactive `products` array

**Tests:** Added `src/pages/__tests__/MarketScreen.test.tsx` covering loading, loaded, and empty states for the investor view.

---

---

### 4. No direct Firestore from frontend — architectural enforcement

**Rule established**: `mombongo-web` must never call Firestore SDK functions (`getDocs`, `getDoc`, `setDoc`, `updateDoc`, `onSnapshot`, etc.) directly. All data reads and writes go through Firebase Cloud Functions (`httpsCallable`). Firebase Auth SDK calls are the only direct Firebase client calls allowed.

**Why**: Security (Firestore rules enforce auth on the backend), auditability, and future flexibility to swap storage layers without changing the frontend.

**What changed**:
- `src/lib/firebase.ts` — removed `db` and `storage` exports (compile-time enforcement)
- `src/hooks/useProducts.ts` — replaced `getDocs(query(collection(db, ...)))` with `httpsCallable(functions, 'getProducts')({})`
- `src/store/AuthContext.tsx` — replaced `getDoc(doc(db, 'users', uid))` with `httpsCallable(functions, 'getUserProfile')({})`
- `src/services/auth.service.ts` — replaced `setDoc(doc(db, 'users', uid), {...})` (in signUp + Google signIn) with `httpsCallable(functions, 'createUserProfile')({...})`
- `CLAUDE.md` — added "Architecture rule — No direct Firestore from frontend" section
- 10 future sprint docs updated — replaced all Firestore SDK patterns in `mombongo-web` sections with `httpsCallable` equivalents

---

## Files changed

| File | Repo | Change |
|------|------|--------|
| `src/test/setup.ts` | mombongo-web | + localStorage mock + react-i18next mock |
| `src/pages/MarketScreen.tsx` | mombongo-web | useProducts hook replacing direct mock import |
| `src/pages/__tests__/MarketScreen.test.tsx` | mombongo-web | New — 3 unit tests |
| `src/lib/firebase.ts` | mombongo-web | Removed `db`, `storage` exports |
| `src/hooks/useProducts.ts` | mombongo-web | Direct Firestore → `httpsCallable(functions, 'getProducts')` |
| `src/store/AuthContext.tsx` | mombongo-web | Direct Firestore getDoc → `httpsCallable(functions, 'getUserProfile')` |
| `src/services/auth.service.ts` | mombongo-web | Direct Firestore setDoc → `httpsCallable(functions, 'createUserProfile')` |
| `CLAUDE.md` | mombongo-web | + no-direct-Firebase architecture rule + updated data hook pattern |
| `sprints/sprint-2-marketplace/S2-01-product-list.md` | sprint-repo | Updated current state |
| `sprints/sprint-2-marketplace/S2-03-product-detail.md` | sprint-repo | useProduct hook → httpsCallable |
| `sprints/sprint-2-marketplace/S2-05-portfolio.md` | sprint-repo | useInvestments hook → httpsCallable |
| `sprints/sprint-3-bourse/S3-01-bourse-screen.md` | sprint-repo | useBourseOpportunities + useBoursePrices → httpsCallable + polling |
| `sprints/sprint-3-bourse/S3-02-bourse-detail.md` | sprint-repo | useBourseOpportunity + usePriceHistory → httpsCallable |
| `sprints/sprint-4-financing/S4-01-financing-screen.md` | sprint-repo | useFarmers → httpsCallable |
| `sprints/sprint-4-financing/S4-02-farmer-detail.md` | sprint-repo | useFarmer + useMyFinancingApplications → httpsCallable |
| `sprints/sprint-4-financing/S4-03-agent-report.md` | sprint-repo | UI-done status + scoped remaining work |
| `sprints/sprint-4-financing/S4-04-cultural-calendar.md` | sprint-repo | useCulturalEvents → httpsCallable |
| `sprints/sprint-5-academia/S5-01-academia-screen.md` | sprint-repo | useCourses → httpsCallable |
| `sprints/sprint-5-academia/S5-02-course-detail.md` | sprint-repo | useCourse + useCourseModules + useMyEnrollment → httpsCallable |
| `sprints/sprint-6-payments-and-notifications/S6-01-payments.md` | sprint-repo | UI-done + PawaPay + Stripe; onSnapshot polling → httpsCallable polling |
| `sprints/sprint-6-payments-and-notifications/S6-02-fcm-notifications.md` | sprint-repo | FCM token updateDoc → registerFcmToken httpsCallable |
| `sprints/sprint-7-pwa-and-production/S7-04-production-deploy.md` | sprint-repo | CinetPay secrets → PawaPay + Stripe secrets |
| `sprints/sprint-7-pwa-and-production/S7-05-production-deploy.md` | sprint-repo | CinetPay post-launch check → PawaPay + Stripe |
| `sprints/sprint-0-infrastructure - DONE/S0-01-repo-setup.md` | sprint-repo | `VITE_CINETPAY_MODE` → `VITE_STRIPE_PUBLISHABLE_KEY` + `VITE_PAWAPAY_ENV` |
| 7 sprint files renamed | sprint-repo | Filename → content alignment |

## Result
- 53/53 tests passing
- 0 TypeScript errors
- `/market` investor view loads from `useProducts` via Cloud Function (`httpsCallable`)
- No direct Firestore SDK calls from any `mombongo-web` frontend code
