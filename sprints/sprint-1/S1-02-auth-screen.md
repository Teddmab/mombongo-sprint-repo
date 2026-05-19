# S1-02 — Auth Screen (Login, Signup, Role Selection)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S1-02 |
| Sprint | Sprint 1 — Auth & Home |
| Branch | `feature/s1-02-auth-screen` |
| Merges into | `dev` |
| Owner | Moïse |
| Estimate | 3 hours |
| Dependencies | S0-01 → S0-04, S1-01 |
| Priority | P0 — Gate for all authenticated features |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | authService + wire form + data-testids + DEV panel |
| `mombongo-admin` | 🔨 Active | Add data-testid to LoginScreen form fields + verify tests |
| `mombongo-mobile` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-functions` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-backoffice` | ⏳ Sprint 2 | Repo not yet initialized |

---

## mombongo-web

### Current State (already implemented — do NOT rewrite)

`src/pages/AuthScreen.tsx` already has a full desktop + mobile UI:
- ✅ Desktop: split-screen (green brand panel left, form panel right)
- ✅ Mobile: green hero top, white card bottom
- ✅ Login form — email + password + eye toggle
- ✅ Role selector (4 role cards: investor / farmer / trader / agent)
- ✅ `useIsDesktop()` responsive layout
- ✅ `data-testid="auth-screen"` on root

**What is NOT wired up yet (this sprint's work):**
- ❌ Submit calls `toast.success` + hardcoded navigate — no real Firebase call
- ❌ No `authService` class
- ❌ No `data-testid` on form fields
- ❌ No error banner on failed login
- ❌ No Google sign-in wired up
- ❌ No DEV_MODE quick-login panel
- ❌ Role type mismatch: screen uses `"trader"` but `AuthContext.UserRole` uses `"merchant"` — align to `"merchant"`

---

## Step 1 — AuthService

Create `src/services/auth.service.ts`. This is a new file — do not touch `AuthScreen.tsx` in this step.

```typescript
import {
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  updateProfile,
  sendPasswordResetEmail,
  signInWithPopup,
  GoogleAuthProvider,
  type User as FirebaseUser,
} from 'firebase/auth'
import { doc, setDoc, serverTimestamp } from 'firebase/firestore'
import { auth, db, isDevMode } from '@/lib/firebase'
import i18n from '@/lib/i18n'
import type { UserRole } from '@/store/AuthContext'

class AuthServiceError extends Error {
  constructor(public code: string, public userMessage: string) { super(userMessage) }
}

const ERROR_MAP: Record<string, string> = {
  'auth/user-not-found':       'auth.error.userNotFound',
  'auth/wrong-password':       'auth.error.wrongPassword',
  'auth/email-already-in-use': 'auth.error.emailInUse',
  'auth/weak-password':        'auth.error.weakPassword',
  'auth/invalid-email':        'auth.error.invalidEmail',
  'auth/network-request-failed': 'auth.error.network',
  'auth/invalid-credential':   'auth.error.wrongPassword',
}

function toAuthError(err: any): AuthServiceError {
  const key = ERROR_MAP[err?.code] ?? 'common.error'
  return new AuthServiceError(err?.code ?? 'unknown', i18n.t(key))
}

class AuthService {
  async signIn(email: string, password: string): Promise<FirebaseUser> {
    if (isDevMode()) {
      // DEV bypass: return a fake user without hitting Firebase
      return { uid: 'dev-user-001', email } as unknown as FirebaseUser
    }
    try {
      const { user } = await signInWithEmailAndPassword(auth, email, password)
      return user
    } catch (e) { throw toAuthError(e) }
  }

  async signUp(
    email: string,
    password: string,
    fullName: string,
    role: UserRole,
  ): Promise<FirebaseUser> {
    try {
      const { user } = await createUserWithEmailAndPassword(auth, email, password)
      await updateProfile(user, { displayName: fullName })
      await setDoc(doc(db, 'users', user.uid), {
        fullName, email, role,
        preferredLanguage: i18n.language as 'fr' | 'en' | 'ln',
        phone: '', avatarUrl: null,
        kycStatus: 'pending', kycVerifiedAt: null,
        mobileMoneyNumber: null, mobileMoneyProvider: null,
        fcmTokens: [], totalInvestedUsd: 0, totalEarnedUsd: 0,
        referralCode: user.uid.slice(-6).toUpperCase(),
        referredBy: null, isActive: true,
        createdAt: serverTimestamp(), updatedAt: serverTimestamp(),
      }, { merge: true })
      return user
    } catch (e) { throw toAuthError(e) }
  }

  async signInWithGoogle(): Promise<FirebaseUser> {
    try {
      const { user } = await signInWithPopup(auth, new GoogleAuthProvider())
      await setDoc(doc(db, 'users', user.uid), {
        fullName: user.displayName ?? '', email: user.email ?? '',
        role: 'investor' as UserRole,
        preferredLanguage: 'fr',
        phone: '', avatarUrl: user.photoURL,
        kycStatus: 'pending', kycVerifiedAt: null,
        mobileMoneyNumber: null, mobileMoneyProvider: null,
        fcmTokens: [], totalInvestedUsd: 0, totalEarnedUsd: 0,
        referralCode: user.uid.slice(-6).toUpperCase(),
        referredBy: null, isActive: true,
        createdAt: serverTimestamp(), updatedAt: serverTimestamp(),
      }, { merge: true })
      return user
    } catch (e) { throw toAuthError(e) }
  }

  async resetPassword(email: string): Promise<void> {
    try { await sendPasswordResetEmail(auth, email) }
    catch (e) { throw toAuthError(e) }
  }
}

export const authService = new AuthService()
export type { AuthServiceError }
```

### Unit Tests
File: `src/services/__tests__/auth.service.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

vi.mock('@/lib/firebase', () => ({
  auth: {},
  db: {},
  isDevMode: () => false,
}))
vi.mock('firebase/auth', () => ({
  signInWithEmailAndPassword: vi.fn(),
  createUserWithEmailAndPassword: vi.fn(),
  updateProfile: vi.fn(),
  sendPasswordResetEmail: vi.fn(),
  signInWithPopup: vi.fn(),
  GoogleAuthProvider: vi.fn(),
}))
vi.mock('firebase/firestore', () => ({
  doc: vi.fn(),
  setDoc: vi.fn(),
  serverTimestamp: vi.fn(),
}))
vi.mock('@/lib/i18n', () => ({ default: { t: (k: string) => k, language: 'fr' } }))

const { authService } = await import('@/services/auth.service')
const { signInWithEmailAndPassword, createUserWithEmailAndPassword, updateProfile } =
  await import('firebase/auth')
const { setDoc } = await import('firebase/firestore')

describe('authService.signIn()', () => {
  beforeEach(() => vi.clearAllMocks())

  it('calls Firebase signInWithEmailAndPassword', async () => {
    vi.mocked(signInWithEmailAndPassword).mockResolvedValue({ user: { uid: 'u1' } } as any)
    await authService.signIn('test@test.com', 'pw123')
    expect(signInWithEmailAndPassword).toHaveBeenCalledWith({}, 'test@test.com', 'pw123')
  })

  it('maps auth/wrong-password to AuthServiceError', async () => {
    vi.mocked(signInWithEmailAndPassword).mockRejectedValue({ code: 'auth/wrong-password' })
    await expect(authService.signIn('x@x.com', 'bad')).rejects.toMatchObject({ code: 'auth/wrong-password' })
  })

  it('maps auth/user-not-found to AuthServiceError', async () => {
    vi.mocked(signInWithEmailAndPassword).mockRejectedValue({ code: 'auth/user-not-found' })
    await expect(authService.signIn('x@x.com', 'pw')).rejects.toMatchObject({ code: 'auth/user-not-found' })
  })

  it('wraps unknown errors with generic message', async () => {
    vi.mocked(signInWithEmailAndPassword).mockRejectedValue({ code: 'auth/unknown-thing' })
    const err: any = await authService.signIn('x@x.com', 'pw').catch(e => e)
    expect(err.userMessage).toBeTruthy()
  })
})

describe('authService.signUp()', () => {
  it('creates user then writes Firestore doc with merge:true', async () => {
    vi.mocked(createUserWithEmailAndPassword).mockResolvedValue({ user: { uid: 'u2', email: 'n@t.com' } } as any)
    vi.mocked(updateProfile).mockResolvedValue(undefined)
    vi.mocked(setDoc).mockResolvedValue(undefined)

    await authService.signUp('n@t.com', 'pw123', 'Jean Test', 'investor')

    expect(createUserWithEmailAndPassword).toHaveBeenCalledOnce()
    expect(setDoc).toHaveBeenCalledWith(
      expect.anything(),
      expect.objectContaining({ role: 'investor', kycStatus: 'pending', totalInvestedUsd: 0 }),
      { merge: true }
    )
  })

  it('does not call setDoc if createUser fails', async () => {
    vi.mocked(createUserWithEmailAndPassword).mockRejectedValue({ code: 'auth/email-already-in-use' })
    await authService.signUp('taken@t.com', 'pw', 'Name', 'investor').catch(() => {})
    expect(setDoc).not.toHaveBeenCalled()
  })
})
```

---

## Step 2 — Wire AuthScreen to authService

**Do not rewrite the existing UI.** Enhance `src/pages/AuthScreen.tsx` in-place:

### Changes required

1. **Add `data-testid` to form fields** (needed for tests):
   - Email input: `data-testid="email-input"`
   - Password input: `data-testid="password-input"`
   - Login submit: `data-testid="login-submit"`
   - Full name input (signup): `data-testid="fullname-input"`
   - Signup submit: `data-testid="signup-submit"`
   - Error banner div: `data-testid="auth-error"`

2. **Replace the mock `submit()` with real auth calls**:
   ```tsx
   const [error, setError] = useState<string | null>(null)
   const [loading, setLoading] = useState(false)

   const handleLogin = async () => {
     setError(null)
     setLoading(true)
     try {
       await authService.signIn(email, pwd)
       navigate('/home')
     } catch (e: any) {
       setError(e.userMessage ?? t('common.error'))
     } finally {
       setLoading(false)
     }
   }
   ```

3. **Error banner** — show when `error` is non-null:
   ```tsx
   {error && (
     <div data-testid="auth-error" className="bg-red-50 border border-red-200 text-red-700 rounded-xl px-4 py-3 text-[13px] flex items-center justify-between">
       {error}
       <button onClick={() => setError(null)}>✕</button>
     </div>
   )}
   ```

4. **Fix role mismatch** — change `"trader"` to `"merchant"` in the roles array to align with `UserRole` in `AuthContext`:
   ```tsx
   // Before
   { id: "trader", emoji: "🏪", key: "auth.roleTrader" }
   // After
   { id: "merchant", emoji: "🏪", key: "auth.roleMerchant" }
   ```
   Update `AppContext.tsx` Role type accordingly: replace `"trader"` with `"merchant"`.

5. **DEV_MODE quick-login panel** — add when `isDevMode()` is true, inside the form card above the inputs:
   ```tsx
   {isDevMode() && (
     <div className="bg-gray-50 border border-gray-200 rounded-xl p-3 mb-4">
       <p className="text-[10px] font-bold text-gray-400 uppercase tracking-wider mb-2">DEV MODE</p>
       <div className="flex flex-wrap gap-2">
         {[
           { label: '👤 Investor', email: 'investor@test.com' },
           { label: '🌾 Farmer',   email: 'farmer@test.com' },
           { label: '🕵️ Agent',   email: 'agent@test.com' },
           { label: '🏪 Merchant', email: 'merchant@test.com' },
         ].map(({ label, email: e }) => (
           <button key={e} type="button"
             onClick={() => { setEmail(e); setPwd('Mombongo2026!'); }}
             className="text-[11px] bg-white border border-gray-200 rounded-lg px-2.5 py-1.5 font-semibold hover:bg-gray-100 transition">
             {label}
           </button>
         ))}
       </div>
     </div>
   )}
   ```

6. **Loading state on button** — disable and show spinner while `loading` is true.

### Unit Tests
File: `src/pages/__tests__/AuthScreen.test.tsx`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { MemoryRouter } from 'react-router-dom'
import AuthScreen from '@/pages/AuthScreen'
import { authService } from '@/services/auth.service'

vi.mock('@/services/auth.service', () => ({
  authService: { signIn: vi.fn(), signUp: vi.fn(), signInWithGoogle: vi.fn(), resetPassword: vi.fn() }
}))
vi.mock('@/context/AppContext', () => ({
  useApp: () => ({ role: 'investor', setRole: vi.fn(), lang: 'fr', setLang: vi.fn(), userName: 'Alain' }),
  AppProvider: ({ children }: any) => children,
}))
const mockNavigate = vi.fn()
vi.mock('react-router-dom', async () => ({
  ...(await vi.importActual('react-router-dom')),
  useNavigate: () => mockNavigate,
}))

describe('AuthScreen — Login', () => {
  beforeEach(() => vi.clearAllMocks())

  it('renders email and password fields', () => {
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    expect(screen.getByTestId('email-input')).toBeInTheDocument()
    expect(screen.getByTestId('password-input')).toBeInTheDocument()
  })

  it('calls authService.signIn with correct args', async () => {
    const user = userEvent.setup()
    vi.mocked(authService.signIn).mockResolvedValue({ uid: 'u1' } as any)
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.clear(screen.getByTestId('email-input'))
    await user.type(screen.getByTestId('email-input'), 'test@test.com')
    await user.clear(screen.getByTestId('password-input'))
    await user.type(screen.getByTestId('password-input'), 'pass123')
    await user.click(screen.getByTestId('login-submit'))
    await waitFor(() => expect(authService.signIn).toHaveBeenCalledWith('test@test.com', 'pass123'))
  })

  it('navigates to /home on successful login', async () => {
    const user = userEvent.setup()
    vi.mocked(authService.signIn).mockResolvedValue({ uid: 'u1' } as any)
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.clear(screen.getByTestId('email-input'))
    await user.type(screen.getByTestId('email-input'), 'test@test.com')
    await user.clear(screen.getByTestId('password-input'))
    await user.type(screen.getByTestId('password-input'), 'pass123')
    await user.click(screen.getByTestId('login-submit'))
    await waitFor(() => expect(mockNavigate).toHaveBeenCalledWith('/home'))
  })

  it('shows error banner on wrong password', async () => {
    const user = userEvent.setup()
    vi.mocked(authService.signIn).mockRejectedValue({ code: 'auth/wrong-password', userMessage: 'Mot de passe incorrect' })
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.clear(screen.getByTestId('email-input'))
    await user.type(screen.getByTestId('email-input'), 'x@x.com')
    await user.clear(screen.getByTestId('password-input'))
    await user.type(screen.getByTestId('password-input'), 'wrong')
    await user.click(screen.getByTestId('login-submit'))
    await waitFor(() => expect(screen.getByTestId('auth-error')).toBeInTheDocument())
  })
})
```

### Regression
```bash
bun run test:unit -- src/services/__tests__/auth.service.test.ts
bun run test:unit -- src/pages/__tests__/AuthScreen.test.tsx
# Expected: 5 + 4 = 9 tests pass
bun run build
```

📝 Manual checklist:
- [ ] Login with email/password → navigates to `/home`
- [ ] Wrong password → red error banner appears
- [ ] DEV MODE panel visible in dev server (VITE_DEV_MODE=true), hidden after `bun run build`
- [ ] Password eye toggle switches input type

---

## mombongo-admin

### Current State (already implemented — do NOT rewrite)

`src/pages/LoginScreen.tsx` already contains a full login UI wired to Firebase:
- ✅ Email + password inputs
- ✅ `signIn(email, password)` via `useAuth()` → `store/AuthContext.tsx` → Firebase
- ✅ Error display via `<p className="error-text">`
- ✅ Loading state (`disabled={loading || isSubmitting}`) on submit button
- ✅ Demo credentials pre-filled (`admin@test.com` / `Mombongo2026!`)
- ✅ `store/AuthContext.tsx` handles DEV demo-mode via `localStorage` session key

**What is NOT done yet:**
- ❌ No `data-testid` on form fields (needed for existing `LoginScreen.test.tsx`)

### Step 1 — Add data-testid to LoginScreen

**Do not change the UI.** Add testid attributes to the three interactive elements in `src/pages/LoginScreen.tsx`:

```tsx
// Email input — add data-testid
<input
  id="email"
  type="email"
  data-testid="email-input"
  value={email}
  onChange={(event) => setEmail(event.target.value)}
  autoComplete="email"
/>

// Password input — add data-testid
<input
  id="password"
  type="password"
  data-testid="password-input"
  value={password}
  onChange={(event) => setPassword(event.target.value)}
  autoComplete="current-password"
/>

// Submit button — add data-testid
<button type="submit" data-testid="login-submit" className="button" disabled={loading || isSubmitting}>

// Error paragraph — add data-testid
{error ? <p data-testid="login-error" className="error-text">{error}</p> : null}
```

### Verify existing tests pass

The test at `src/pages/__tests__/LoginScreen.test.tsx` already exists and tests:
- Successful sign-in navigates to `/admin`
- Error message renders on failed sign-in

```bash
# Run from mombongo-admin/
bun run test:unit
# Expected: all existing tests pass
bun run build
# Expected: exits 0
```

📝 Manual checklist:
- [ ] Login with `admin@test.com` / `Mombongo2026!` → navigates to `/admin`
- [ ] Wrong password → error text appears below form
- [ ] Submit button disabled while login is in progress

---

## ✅ Milestone — S1-02 Complete
- [ ] **[web]** 9 unit tests pass (5 authService + 4 AuthScreen)
- [ ] **[web]** `authService` in `src/services/auth.service.ts`
- [ ] **[web]** Real Firebase login works when VITE_DEV_MODE=false
- [ ] **[web]** Role type aligned: `"merchant"` not `"trader"` everywhere
- [ ] **[web]** DEV_MODE panel absent in production build
- [ ] **[admin]** `data-testid` on email input, password input, submit button, error paragraph
- [ ] **[admin]** All existing admin tests pass
- [ ] **[admin]** `bun run build` exits 0

## 🏁 PR Checklist (SECURITY CRITICAL)
- [ ] `bun run test:ci` exits 0 (both web and admin)
- [ ] No Firebase auth calls outside `auth.service.ts` (web) / `store/AuthContext.tsx` (admin)
- [ ] `setDoc` uses `{ merge: true }` (web signUp)
- [ ] Afrotouch OU review required

```bash
# mombongo-web
git add -A
git commit -m "feat(s1-02): authService + wire login/signup to Firebase, DEV_MODE panel"
git push origin feature/s1-02-auth-screen

# mombongo-admin
git add -A
git commit -m "feat(s1-02): add data-testid to LoginScreen form fields"
git push origin feature/s1-02-admin-login
```
