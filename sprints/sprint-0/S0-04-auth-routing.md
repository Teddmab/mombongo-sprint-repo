# S0-04 — AuthProvider, Context & Route Structure

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S0-04 |
| Sprint | Sprint 0 — Infrastructure |
| Branch | `feature/s0-04-auth-routing` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 2 hours |
| Dependencies | S0-01, S0-02, S0-03 |

---

## Step 1 — AuthProvider + Hooks

### Lovable Prompt
```
Create src/store/AuthContext.tsx:

Interface AuthContextValue:
  user: FirebaseUser | null
  userProfile: UserProfile | null
  isLoading: boolean
  isAuthenticated: boolean
  signOut: () => Promise<void>

AuthProvider component:
- onAuthStateChanged(auth, async (firebaseUser) => {
    if (firebaseUser) {
      try {
        const snap = await getDoc(doc(db, 'users', firebaseUser.uid))
        setUserProfile(snap.exists() ? snap.data() as UserProfile : null)
      } catch { setUserProfile(null) }
      setUser(firebaseUser)
    } else {
      setUser(null); setUserProfile(null)
    }
    setIsLoading(false)
  })
- Return unsubscribe on unmount.
- While isLoading: render <LoadingScreen />.
- DEV_MODE (isDevMode() === true): skip Firebase, inject mock investor user.
- signOut: calls signOut(auth) from firebase/auth.

Create src/hooks/useAuth.ts:
export const useAuth = () => {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth must be within AuthProvider')
  return ctx
}

Create src/hooks/useRole.ts:
Returns from useAuth().userProfile?.role:
  isAdmin, isAgent, isFarmer, isMerchant, isInvestor (all boolean)
  role: UserRole | null
  canInvest: role === 'investor' || role === 'merchant' || role === 'admin'
  canSubmitReport: role === 'agent' || role === 'admin'
  canAccessAdmin: role === 'admin'

Create src/components/layout/ProtectedRoute.tsx:
If isLoading: <SkeletonLoader variant="card" />
If !isAuthenticated: <Navigate to="/language" replace />
Else: render children

Create src/components/layout/AdminRoute.tsx:
const { canAccessAdmin } = useRole()
If !canAccessAdmin: <Navigate to="/home" replace />
Else: render children

Create src/pages/LoadingScreen.tsx:
Full screen, bg-[#1E6B3F], centered:
  "🌿" (64px) + "Mombongo" (white, 28px, bold) + pulsing ring below.
  data-testid="loading-screen"
```

---

## Step 2 — Routes + Stub Screens

### Lovable Prompt
```
Update src/main.tsx to wrap the app in this provider order:
1. QueryClientProvider — queryClient config:
   staleTime: 300000 (5 min), retry: 2, refetchOnWindowFocus: false
2. Suspense fallback={<LoadingScreen />}
3. AuthProvider
4. BrowserRouter
5. AppRoutes (new component)

Import i18n from '@/lib/i18n' at the top of main.tsx (side-effect import).

Create src/AppRoutes.tsx with React.lazy routes:

Public (no auth):
  /language → LanguageScreen
  /auth → AuthScreen

Protected (inside ProtectedRoute):
  /home → HomeScreen
  /market → MarketScreen
  /market/:id → ProductDetailScreen
  /invest/:id → InvestScreen
  /bourse → BourseScreen
  /bourse/:id → BourseDetailScreen
  /financing → FinancingScreen
  /financing/:id → FarmerDetailScreen
  /report/new → AgentReportScreen
  /academia → AcademiaScreen
  /academia/:id → CourseDetailScreen
  /academia/:courseId/module/:moduleId → ModulePlayerScreen
  /profile → ProfileScreen
  /portfolio → PortfolioScreen
  /notifications → NotificationsScreen

Admin (inside ProtectedRoute + AdminRoute):
  /admin → AdminDashboard
  /admin/users → AdminUsers
  /admin/transactions → AdminTransactions
  /admin/bourse → AdminBourse
  /admin/financing → AdminFinancing

Index: / redirects to /language

Create stub screen for every route above. Each stub:
const ScreenName = () => <div data-testid="screen-name-kebab">ScreenName</div>
export default ScreenName

Files:
src/pages/LanguageScreen.tsx — data-testid="language-screen"
src/pages/AuthScreen.tsx — data-testid="auth-screen"
src/pages/HomeScreen.tsx — data-testid="home-screen"
src/pages/MarketScreen.tsx — data-testid="market-screen"
src/pages/ProductDetailScreen.tsx — data-testid="product-detail-screen"
src/pages/InvestScreen.tsx — data-testid="invest-screen"
src/pages/BourseScreen.tsx — data-testid="bourse-screen"
src/pages/BourseDetailScreen.tsx — data-testid="bourse-detail-screen"
src/pages/FinancingScreen.tsx — data-testid="financing-screen"
src/pages/FarmerDetailScreen.tsx — data-testid="farmer-detail-screen"
src/pages/AgentReportScreen.tsx — data-testid="agent-report-screen"
src/pages/AcademiaScreen.tsx — data-testid="academia-screen"
src/pages/CourseDetailScreen.tsx — data-testid="course-detail-screen"
src/pages/ModulePlayerScreen.tsx — data-testid="module-player-screen"
src/pages/ProfileScreen.tsx — data-testid="profile-screen"
src/pages/PortfolioScreen.tsx — data-testid="portfolio-screen"
src/pages/NotificationsScreen.tsx — data-testid="notifications-screen"
src/pages/admin/AdminDashboard.tsx — data-testid="admin-dashboard"
src/pages/admin/AdminUsers.tsx — data-testid="admin-users"
src/pages/admin/AdminTransactions.tsx — data-testid="admin-transactions"
src/pages/admin/AdminBourse.tsx — data-testid="admin-bourse"
src/pages/admin/AdminFinancing.tsx — data-testid="admin-financing"
```

### Regression — Sprint 0 Final Gate
```bash
bun run typecheck   # ✅ exits 0
bun run lint        # ✅ exits 0
bun run test:unit   # ✅ passes
bun run build       # ✅ exits 0 with 20+ chunks in dist/assets/
```

📝 Manual: open every route in browser — all show stub content, no crashes.
📝 Manual: navigate to /admin as investor → redirected to /home.
📝 Manual: navigate to /market when logged out → redirected to /language.

## ✅ Sprint 0 Complete 🎉
All infrastructure in place. Sprints 1-7 build features on top of this.

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0
- [ ] `bun run build` exits 0 with 20+ code-split chunks
- [ ] All 22 stub screen files created with correct data-testid
- [ ] QueryClient: refetchOnWindowFocus: false (DRC network UX)
- [ ] AuthProvider unsubscribes onAuthStateChanged on unmount
- [ ] DEV_MODE bypasses Firebase (verify: no Firebase network calls in DevTools)

```bash
git add -A
git commit -m "feat(s0-04): auth provider, useAuth, useRole, protected routes, all stub screens"
git push origin feature/s0-04-auth-routing
# Open PR → dev
```
