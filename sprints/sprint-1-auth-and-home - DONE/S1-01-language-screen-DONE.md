# S1-01 — Language Selection Screen

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S1-01 |
| Sprint | Sprint 1 — Auth & Home |
| Branch | `feature/s1-01-language-screen` |
| Merges into | `dev` |
| Owner | Moïse |
| Estimate | 0.5 hours |
| Dependencies | S0-03 (i18n), S0-04 (routing) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Add `data-testid` to language buttons |
| `mombongo-admin` | ✅ N/A | Admin has no language selection — internal tool, always FR |
| `mombongo-mobile` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-functions` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-backoffice` | ⏳ Sprint 2 | Repo not yet initialized |

---

## mombongo-web

### Current State (already implemented — do NOT rewrite)

`src/pages/LanguageSelection.tsx` already contains a full desktop + mobile implementation:
- ✅ Green gradient background, Mombongo logo, rotating tagline every 3 s (Framer Motion)
- ✅ 3 language buttons (FR / EN / LN) with stagger entrance animation
- ✅ `useApp().setLang(lang)` on press → persists to `localStorage` under key `mb_lang` via `AppContext`
- ✅ `setTimeout(120ms)` → `navigate('/auth')`
- ✅ Desktop split-panel variant via `useIsDesktop()`
- ✅ `data-testid="language-screen"` on root element

**File name is `LanguageSelection.tsx`, not `LanguageScreen.tsx`. Keep this name — do not rename.**

### Step 1 — Add data-testid to language buttons

The only missing piece for testability: `data-testid` attributes on the three buttons.

**Mobile buttons** (inside `.space-y-3` list):
```tsx
// Before
<button key={l.code} onClick={() => choose(l.code)} ...>

// After — add data-testid
<button key={l.code} data-testid={`language-btn-${l.code}`} onClick={() => choose(l.code)} ...>
```

**Desktop buttons** (inside `.grid.grid-cols-3` grid):
```tsx
// Before
<button key={l.code} onClick={() => choose(l.code)} ...>

// After — add data-testid
<button key={l.code} data-testid={`language-btn-${l.code}`} onClick={() => choose(l.code)} ...>
```

### Unit Tests
File: `src/pages/__tests__/LanguageScreen.test.tsx`

> **Note:** test imports `LanguageSelection` from `@/pages/LanguageSelection` (existing filename).

```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { MemoryRouter } from 'react-router-dom'
import LanguageSelection from '@/pages/LanguageSelection'

const mockNavigate = vi.fn()
vi.mock('react-router-dom', async () => ({
  ...(await vi.importActual('react-router-dom')),
  useNavigate: () => mockNavigate,
}))

vi.mock('@/context/AppContext', () => ({
  useApp: () => ({ setLang: vi.fn(), lang: 'fr', role: 'investor', userName: 'Alain' }),
  AppProvider: ({ children }: any) => children,
}))

describe('LanguageSelection', () => {
  it('renders 3 language buttons', () => {
    render(<MemoryRouter><LanguageSelection /></MemoryRouter>)
    expect(screen.getAllByTestId(/language-btn-/)).toHaveLength(3)
  })

  it('renders FR, EN and LN buttons', () => {
    render(<MemoryRouter><LanguageSelection /></MemoryRouter>)
    expect(screen.getByTestId('language-btn-fr')).toBeInTheDocument()
    expect(screen.getByTestId('language-btn-en')).toBeInTheDocument()
    expect(screen.getByTestId('language-btn-ln')).toBeInTheDocument()
  })

  it('shows Mombongo branding', () => {
    render(<MemoryRouter><LanguageSelection /></MemoryRouter>)
    expect(screen.getByAltText(/Mombongo/i)).toBeInTheDocument()
  })

  it('selecting FR calls setLang and eventually navigates to /auth', async () => {
    vi.useFakeTimers()
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime })
    render(<MemoryRouter><LanguageSelection /></MemoryRouter>)
    await user.click(screen.getByTestId('language-btn-fr'))
    vi.advanceTimersByTime(200)
    expect(mockNavigate).toHaveBeenCalledWith('/auth')
    vi.useRealTimers()
  })
})
```

### Regression
```bash
# Run from mombongo-web/
bun run test:unit -- src/pages/__tests__/LanguageScreen.test.tsx
# Expected: 4 tests pass
bun run build
# Expected: exits 0
```

📝 Manual: `/language` → click any button → navigates to `/auth`.

---

## mombongo-admin

No language selection screen in admin — it is an internal operations tool always used in French. The admin app routes directly to `/login` on cold load.

✅ **No changes required for this story.**

---

## ✅ Milestone — S1-01 Complete
- [ ] **[web]** 4 unit tests pass
- [ ] **[web]** `data-testid="language-btn-fr/en/ln"` present on all 3 buttons (mobile + desktop)
- [ ] **[web]** Language saves to `localStorage` key `mb_lang` on selection
- [ ] **[web]** Navigates to `/auth` after ~120ms
- [ ] **[admin]** No changes needed — already complete

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0 (web)
- [ ] `bun run build` exits 0 (web)
- [ ] No rename of `LanguageSelection.tsx`

```bash
# mombongo-web
git add -A
git commit -m "feat(s1-01): add data-testid to language buttons + unit tests"
git push origin feature/s1-01-language-screen
```
