# S7-02 — Polish — i18n Completeness, Animations & Accessibility

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S7-02 |
| Sprint | Sprint 7 — PWA & Production |
| Branch | `feature/s7-02-polish` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | All feature sprints complete |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | Translation audit, aria-labels, focus management, animation consistency |
| `mombongo-admin` | 🔨 Active | Translation audit, aria-labels |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — i18n completeness audit

Run a script to find missing translation keys:

```bash
# Find all t('...') usages in src and check against fr.json / en.json
grep -roh "t('[^']*')" src/ | sort -u | sed "s/t('//;s/'$//" > /tmp/used_keys.txt
jq -r 'paths | join(".")' src/i18n/fr.json > /tmp/fr_keys.txt
comm -23 <(sort /tmp/used_keys.txt) <(sort /tmp/fr_keys.txt)
```

For every missing key:
1. Add French translation to `src/i18n/fr.json`
2. Add English translation to `src/i18n/en.json`
3. Use consistent format: `namespace.feature.key`

**Known namespaces to complete**: `common`, `home`, `market`, `invest`, `bourse`, `financing`, `academia`, `portfolio`, `deposit`, `notifications`, `pwa`, `agent`, `calendar`

### Step 2 — Aria-labels on interactive elements

Priority list — add `aria-label` to any icon-only buttons:

```tsx
// BottomNav tab buttons
<button aria-label={t('nav.home')} ...>

// Language toggle
<button aria-label={t('common.switchLang')} ...>

// Close/dismiss buttons
<button aria-label={t('common.close')} ...>

// Investment + fund CTAs
<button aria-label={t('invest.openModal', { product: product.name })} ...>

// Notification bell
<button aria-label={t('notifications.enable')} ...>
```

### Step 3 — Focus management in modals

All modal components (`InvestModal`, `BourseInvestModal`, `FundModal`, `DepositModal`) must:

1. Focus the first focusable element on open:
```typescript
useEffect(() => {
  if (isOpen) {
    const firstInput = modalRef.current?.querySelector<HTMLElement>(
      'input, button, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    firstInput?.focus()
  }
}, [isOpen])
```

2. Trap focus within the modal while open (use `focus-trap-react` or manual implementation).

3. Return focus to the trigger button on close:
```typescript
const triggerRef = useRef<HTMLButtonElement>(null)
// ... when modal closes:
triggerRef.current?.focus()
```

4. Support `Escape` key to dismiss:
```typescript
useEffect(() => {
  const handler = (e: KeyboardEvent) => { if (e.key === 'Escape') onClose() }
  document.addEventListener('keydown', handler)
  return () => document.removeEventListener('keydown', handler)
}, [onClose])
```

### Step 4 — Animation consistency

Ensure all page transitions use the same Framer Motion preset. Create `src/lib/motionPresets.ts`:

```typescript
import type { Variants } from 'framer-motion'

export const pageVariants: Variants = {
  initial: { opacity: 0, y: 12 },
  animate: { opacity: 1, y: 0, transition: { duration: 0.2, ease: 'easeOut' } },
  exit:    { opacity: 0, y: -8, transition: { duration: 0.15 } },
}

export const modalVariants: Variants = {
  initial: { opacity: 0, scale: 0.95, y: 20 },
  animate: { opacity: 1, scale: 1,    y: 0,  transition: { duration: 0.2, ease: 'easeOut' } },
  exit:    { opacity: 0, scale: 0.95, y: 20, transition: { duration: 0.15 } },
}

export const slideUpVariants: Variants = {
  initial: { opacity: 0, y: 40 },
  animate: { opacity: 1, y: 0,  transition: { duration: 0.25, ease: 'easeOut' } },
  exit:    { opacity: 0, y: 40, transition: { duration: 0.15 } },
}
```

Apply `pageVariants` to every screen component that uses `<motion.div>`.

### Step 5 — Skeleton loading consistency

Ensure every data-fetching screen shows `SkeletonLoader` while `isLoading === true`:
- `HomeScreen` ✓ (done in S1-03)
- `MarketScreen` — verify skeleton shown during `useFeaturedProducts` load
- `BourseScreen` — add skeleton for opportunities list
- `FinancingScreen` — add skeleton for farmer cards
- `AcademiaScreen` — add skeleton for course cards
- `PortfolioScreen` — add skeleton for investment list

`SkeletonLoader` reusable pattern:
```tsx
function CardSkeleton() {
  return (
    <div className="rounded-2xl bg-gray-100 animate-pulse h-28 w-full" />
  )
}
```

---

## mombongo-admin

### Accessibility pass

Same focus management and aria-label audit for admin screens. At minimum:
- All table action buttons: `aria-label="Modifier {name}"`, `aria-label="Supprimer {name}"`
- All drawer close buttons: `aria-label="Fermer"`
- Form inputs: all have associated `<label>` elements

---

## ✅ Definition of Done
- [ ] No missing keys in `fr.json` or `en.json`
- [ ] All icon-only buttons have `aria-label`
- [ ] All modals trap focus and close on Escape
- [ ] All screens use `pageVariants` for entry animation
- [ ] All screens show skeleton while loading
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s7-02): i18n completeness, accessibility, animation polish"
git push origin feature/s7-02-polish
```
