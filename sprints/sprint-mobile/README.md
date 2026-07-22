# Mombongo Mobile — Sprint Plan

React Native + Expo app (`mombongo-mobile`). Firebase Auth + Cloud Functions backend. EAS Build for Android (iOS TBD).

## Sprint index

| Sprint | Folder | Focus |
|--------|--------|-------|
| SM-00 | SM-00-infrastructure | CI lint, branch conventions, iOS EAS profile |
| SM-01 | SM-01-data-layer | Hook audit, transactions hook, cultural calendar CF |
| SM-02 | SM-02-financing | Cultural calendar screen, agent farmer assignment |
| SM-03 | SM-03-academia | Module player wiring, certificate unlock |
| SM-04 | SM-04-kyc-upload | Camera/gallery KYC doc upload + status |
| SM-05 | SM-05-push-notifications | expo-notifications + FCM real notifications screen |
| SM-06 | SM-06-agro-exchange | New AgroExchange tab (commodity market) |
| SM-07 | SM-07-subscription | Subscription plans + premium content gating |
| SM-08 | SM-08-payments | Stripe card path + PawaPay UX polish |
| SM-09 | SM-09-offline-performance | React Query AsyncStorage persister, image caching |
| SM-10 | SM-10-monitoring | Sentry RN + Firebase Analytics events |
| SM-11 | SM-11-ios | EAS iOS profiles, safe-area fixes, haptics |

## Branch convention
- Feature: `feature/sm-NN-slug` (e.g., `feature/sm-01-data-layer`)
- Patch: `feature/smp-NN-slug`
- PRs merge into `dev`; `dev → main` triggers EAS preview build

## DONE convention
When a story is fully implemented and verified, rename the file by appending `-DONE` before `.md`.
`SM-00-00-ci-lint.md` → `SM-00-00-ci-lint-DONE.md`

## Gap analysis (web vs mobile — as of 2026-07-22)

### Confirmed working on mobile (matches web)
- Auth: email/password + Google sign-in (popup/GoogleSignInButton)
- Home: all 4 roles (investor, farmer, agent, merchant)
- Market: all 4 roles + ProductDetail + InvestModal
- Bourse: all 4 roles + BourseDetail + invest flow
- Financement: all 4 roles + FarmerDetail + AgentReport
- Portfolio: PortfolioScreen + useInvestments (httpsCallable)
- Academia: AcademiaScreen + CourseDetailScreen
- Profile: ProfileScreen + wallet balance display
- Wallet: WalletModals → wallet.service.ts → Cloud Functions (deposit/withdraw)
- PaymentModal: multi-step (amount → method → details → processing → success) with mobile money operators
- Language selection screen

### Confirmed gaps vs web
| Gap | Sprint |
|-----|--------|
| ESLint + Jest missing from CI | SM-00 |
| iOS EAS build profile | SM-11 |
| `useTransactions` hook missing (no tx history screen) | SM-01 |
| `useBourse` ticker — verify real CF path | SM-01 |
| Market filter sheet — less complete than web (missing crop/harvest date) | SM-01 |
| Cultural calendar in financing | SM-02 |
| Agent terrain — farmer assignment scheduling | SM-02 |
| ModulePlayerModal — not wired to real progress tracking | SM-03 |
| CertificatePreviewModal — not wired to completion gate | SM-03 |
| KYC document upload (camera / gallery → signed URL CF) | SM-04 |
| Push notifications (expo-notifications + FCM token) | SM-05 |
| NotificationsScreen uses local storage only | SM-05 |
| AgroExchange screen does not exist | SM-06 |
| SubscriptionModal — not wired to real plans or payment CF | SM-07 |
| Premium content gating in Academia | SM-07 |
| Stripe card payment path (PaymentModal "card" method is stub) | SM-08 |
| React Query persistence (no offline cache) | SM-09 |
| Sentry React Native | SM-10 |
| Firebase Analytics screen_view + business events | SM-10 |
