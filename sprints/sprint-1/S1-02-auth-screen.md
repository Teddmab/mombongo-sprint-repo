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
| Milestone | Day Off 2 — M1 (Demo Ready) |
| Priority | P0 — Gate for all authenticated features |

---

## Step 1 — AuthService

### Lovable Prompt
```
Create src/services/auth.service.ts.

class AuthServiceError extends Error {
  constructor(public code: string, public userMessage: string) { super(userMessage) }
}

const ERROR_MAP: Record<string, string> = {
  'auth/user-not-found': 'auth.error.userNotFound',
  'auth/wrong-password': 'auth.error.wrongPassword',
  'auth/email-already-in-use': 'auth.error.emailInUse',
  'auth/weak-password': 'auth.error.weakPassword',
  'auth/invalid-email': 'auth.error.invalidEmail',
  'auth/network-request-failed': 'auth.error.network',
  'auth/invalid-credential': 'auth.error.wrongPassword',
}

function toAuthError(err: any): AuthServiceError {
  const key = ERROR_MAP[err?.code] ?? 'common.error'
  return new AuthServiceError(err?.code ?? 'unknown', i18n.t(key))
}

class AuthService {
  async signIn(email: string, password: string): Promise<FirebaseUser> {
    try {
      const { user } = await signInWithEmailAndPassword(auth, email, password)
      return user
    } catch (e) { throw toAuthError(e) }
  }

  async signUp(email: string, password: string, fullName: string, role: UserRole): Promise<FirebaseUser> {
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
        role: 'investor', preferredLanguage: 'fr',
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
import { authService } from '@/services/auth.service'

const { signInWithEmailAndPassword, createUserWithEmailAndPassword, updateProfile } =
  await import('firebase/auth')
const { setDoc } = await import('firebase/firestore')

describe('authService.signIn()', () => {
  beforeEach(() => vi.clearAllMocks())

  it('calls Firebase signInWithEmailAndPassword', async () => {
    vi.mocked(signInWithEmailAndPassword).mockResolvedValue({ user: { uid: 'u1' } } as any)
    await authService.signIn('test@test.com', 'pw123')
    expect(signInWithEmailAndPassword).toHaveBeenCalledWith(expect.anything(), 'test@test.com', 'pw123')
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

### Regression
```bash
bun run test:unit -- src/services/__tests__/auth.service.test.ts
# Expected: 5 tests pass
```

---

## Step 2 — Auth Screen UI

### Lovable Prompt
```
Replace AuthScreen stub with full implementation.

No AppShell. Standalone full-screen.

TOP SECTION (30% height, bg-[#1E6B3F]):
  Large 🌿 emoji + "Mombongo" white bold text (centered)
  Rotating tagline (changes every 3s, fade transition):
    FR: "Investis dans l'avenir du Congo"
    EN: "Invest in Congo's future"
    LN: "Tia mbongo na kala ya Congo"

BOTTOM CARD (70% height, bg-white, rounded-t-3xl, -mt-6, overlapping green):
  shadcn Tabs: "Se connecter" | "S'inscrire"

  LOGIN TAB:
    Email input (type=email, autocomplete=email, placeholder=t('auth.email'), data-testid="email-input")
    Password input (type=password, data-testid="password-input") + Eye toggle button
    "Mot de passe oublié?" link → opens ForgotPasswordSheet
    Submit button: full-width green, t('auth.login'), data-testid="login-submit"
    Google button: outlined, Google SVG icon, "Continuer avec Google"
    Error banner: red bg, dismissible, data-testid="auth-error"

  SIGNUP TAB:
    Full name input (data-testid="fullname-input")
    Email input
    Password input + Eye toggle
    Role selector — 3 cards:
      🌱 t('auth.roleInvestor') — data-role="investor" data-selected="true/false"
      👨‍🌾 t('auth.roleFarmer') — data-role="farmer"
      🏪 t('auth.roleMerchant') — data-role="merchant"
      Selected: border-2 border-[#1E6B3F] bg-[#EBF5EE], checkmark icon top-right
      Default selected: investor
    Submit button: full-width green, t('auth.signup'), data-testid="signup-submit"

  ForgotPasswordSheet (shadcn Sheet from bottom):
    Email input + "Envoyer le lien" button
    On success: show green checkmark + "Email envoyé!"

Validation on submit (not on change):
  email: must include @ and .
  password: min 6 chars
  fullName: min 2 chars (signup)
  Show field error below each invalid field in red text.

On successful login/signup:
  Show spinner on button (loading state)
  Call authService.signIn() or authService.signUp()
  navigate('/home') on success
  Show error banner on AuthServiceError

DEV_MODE section (hidden in prod, visible when VITE_DEV_MODE=true):
  Small gray banner at top of card: "DEV MODE"
  5 quick-login buttons:
    "👤 Investor" → fills investor@test.com / Mombongo2026! → auto-submits
    "🌾 Farmer", "🕵️ Agent", "⚙️ Admin", "🏪 Merchant"
```

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
    await user.type(screen.getByTestId('email-input'), 'test@test.com')
    await user.type(screen.getByTestId('password-input'), 'pass123')
    await user.click(screen.getByTestId('login-submit'))
    await waitFor(() => expect(authService.signIn).toHaveBeenCalledWith('test@test.com', 'pass123'))
  })

  it('navigates to /home on success', async () => {
    const user = userEvent.setup()
    vi.mocked(authService.signIn).mockResolvedValue({ uid: 'u1' } as any)
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.type(screen.getByTestId('email-input'), 'test@test.com')
    await user.type(screen.getByTestId('password-input'), 'pass123')
    await user.click(screen.getByTestId('login-submit'))
    await waitFor(() => expect(mockNavigate).toHaveBeenCalledWith('/home'))
  })

  it('shows error banner on wrong password', async () => {
    const user = userEvent.setup()
    vi.mocked(authService.signIn).mockRejectedValue({ code: 'auth/wrong-password', userMessage: 'Mot de passe incorrect' })
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.type(screen.getByTestId('email-input'), 'x@x.com')
    await user.type(screen.getByTestId('password-input'), 'wrong')
    await user.click(screen.getByTestId('login-submit'))
    await waitFor(() => expect(screen.getByTestId('auth-error')).toBeInTheDocument())
  })

  it('password toggle changes input type', async () => {
    const user = userEvent.setup()
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    expect(screen.getByTestId('password-input')).toHaveAttribute('type', 'password')
    await user.click(screen.getByRole('button', { name: /show|hide|eye/i }))
    expect(screen.getByTestId('password-input')).toHaveAttribute('type', 'text')
  })
})

describe('AuthScreen — Signup', () => {
  it('shows role selector with investor selected by default', async () => {
    const user = userEvent.setup()
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.click(screen.getByRole('tab', { name: /inscrire/i }))
    expect(screen.getByTestId('role-investor') ?? screen.getByText(/investisseur/i)).toBeInTheDocument()
  })

  it('calls signUp with farmer role when selected', async () => {
    const user = userEvent.setup()
    vi.mocked(authService.signUp).mockResolvedValue({ uid: 'u2' } as any)
    render(<MemoryRouter><AuthScreen /></MemoryRouter>)
    await user.click(screen.getByRole('tab', { name: /inscrire/i }))
    await user.type(screen.getByTestId('fullname-input'), 'Jean Test')
    await user.type(screen.getByTestId('email-input'), 'jean@test.com')
    await user.type(screen.getByTestId('password-input'), 'pass123')
    await user.click(screen.getByText(/agriculteur/i))
    await user.click(screen.getByTestId('signup-submit'))
    await waitFor(() =>
      expect(authService.signUp).toHaveBeenCalledWith('jean@test.com', 'pass123', 'Jean Test', 'farmer')
    )
  })
})
```

### Regression
```bash
bun run test:unit -- src/services/__tests__/auth.service.test.ts
bun run test:unit -- src/pages/__tests__/AuthScreen.test.tsx
# Expected: 5 + 6 = 11 tests pass
bun run build
```

📝 Manual checklist:
- [ ] Login investor@test.com / Mombongo2026! → navigates to /home
- [ ] Wrong password → red error banner
- [ ] Signup → new account created in Firebase Console
- [ ] DEV MODE quick-login buttons work (VITE_DEV_MODE=true)
- [ ] Password eye toggle works

---

## ✅ Milestone — S1-02 Complete
- [ ] 11 unit tests pass
- [ ] Real Firebase login works on staging
- [ ] AuthService.signUp uses setDoc with merge:true

## 🏁 PR Checklist (SECURITY CRITICAL)
- [ ] `bun run test:ci` exits 0
- [ ] No Firebase calls outside auth.service.ts (grep check)
- [ ] setDoc uses { merge: true }
- [ ] DEV_MODE section absent in production build
- [ ] Afrotouch OU review required

```bash
git add -A
git commit -m "feat(s1-02): auth service, login/signup screen with role selector"
git push origin feature/s1-02-auth-screen
```
