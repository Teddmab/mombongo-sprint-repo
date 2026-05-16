# S1-01 — Language Selection Screen

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S1-01 |
| Sprint | Sprint 1 — Auth & Home |
| Branch | `feature/s1-01-language-screen` |
| Merges into | `dev` |
| Owner | Moïse |
| Estimate | 1.5 hours |
| Dependencies | S0-03 (i18n), S0-04 (routing) |
| Milestone | Day Off 2 |

---

## Step 1 — Language Screen UI

### Lovable Prompt
```
Replace the LanguageScreen stub with a full implementation.

No AppShell. Full-screen standalone.
Background: linear-gradient(180deg, #1E6B3F 0%, #155130 60%, #0F3D23 100%)
Vertically center all content.

Top section:
  🌿 emoji (80px font-size)
  "MOMBONGO" text (white, 36px, font-bold, tracking-widest, mt-4)
  "Coopérative Agricole Numérique" (white, 14px, opacity-80, mt-2)

Three language buttons (mt-16, flex flex-col gap-4):
  Each button: w-full, rounded-2xl, py-5 px-6
  Background: white with opacity-10, hover opacity-20
  Layout: flex justify-between items-center
  Left: flag emoji (text-3xl) + language code (text-white font-bold ml-3)
  Right: native language name (text-white text-sm) + → arrow

  Button 1: data-testid="language-btn-fr" — 🇫🇷 FR — Français
  Button 2: data-testid="language-btn-en" — 🇬🇧 EN — English
  Button 3: data-testid="language-btn-ln" — 🇨🇩 LN — Lingala

On button press:
  1. i18n.changeLanguage(lang)
  2. localStorage.setItem('mombongo-language', lang)
  3. Framer Motion: animate selected button scale 0.95 → 1.05 → 1
  4. setTimeout 300ms → navigate('/auth')

Entrance animation (Framer Motion):
  Each button: initial={opacity:0, y:60} animate={opacity:1, y:0}
  Stagger delay: index * 0.1s
```

### Unit Tests
File: `src/pages/__tests__/LanguageScreen.test.tsx`
```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { MemoryRouter } from 'react-router-dom'
import LanguageScreen from '@/pages/LanguageScreen'

const mockNavigate = vi.fn()
vi.mock('react-router-dom', async () => ({
  ...(await vi.importActual('react-router-dom')),
  useNavigate: () => mockNavigate,
}))

describe('LanguageScreen', () => {
  it('renders 3 language buttons', () => {
    render(<MemoryRouter><LanguageScreen /></MemoryRouter>)
    expect(screen.getByTestId('language-btn-fr')).toBeInTheDocument()
    expect(screen.getByTestId('language-btn-en')).toBeInTheDocument()
    expect(screen.getByTestId('language-btn-ln')).toBeInTheDocument()
  })

  it('shows MOMBONGO branding', () => {
    render(<MemoryRouter><LanguageScreen /></MemoryRouter>)
    expect(screen.getByText('MOMBONGO')).toBeInTheDocument()
  })

  it('selecting FR saves to localStorage and navigates', async () => {
    const user = userEvent.setup()
    render(<MemoryRouter><LanguageScreen /></MemoryRouter>)
    await user.click(screen.getByTestId('language-btn-fr'))
    expect(localStorage.getItem('mombongo-language')).toBe('fr')
    await vi.waitFor(() => expect(mockNavigate).toHaveBeenCalledWith('/auth'))
  })

  it('selecting LN saves Lingala to localStorage', async () => {
    const user = userEvent.setup()
    render(<MemoryRouter><LanguageScreen /></MemoryRouter>)
    await user.click(screen.getByTestId('language-btn-ln'))
    expect(localStorage.getItem('mombongo-language')).toBe('ln')
  })
})
```

### Regression
```bash
bun run test:unit -- src/pages/__tests__/LanguageScreen.test.tsx
# Expected: 4 tests pass
bun run build
```

📝 Manual: `/language` → click FR → navigates to `/auth`.

---

## ✅ Milestone — S1-01 Complete
- [ ] 4 unit tests pass
- [ ] Language saves to localStorage on selection
- [ ] Stagger animation plays on mount
- [ ] Navigates to /auth after 300ms

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0
- [ ] No hardcoded strings

```bash
git add -A
git commit -m "feat(s1-01): language selection screen with stagger animation"
git push origin feature/s1-01-language-screen
```
