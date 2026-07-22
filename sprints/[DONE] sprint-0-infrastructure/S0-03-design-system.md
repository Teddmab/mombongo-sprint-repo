# S0-03 — Design System, i18n & Layout Components

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S0-03 |
| Sprint | Sprint 0 — Infrastructure |
| Branch | `feature/s0-03-design-system` |
| Merges into | `dev` |
| Owner | Moïse |
| Estimate | 2.5 hours |
| Dependencies | S0-01 |

---

## Step 1 — CSS Tokens + i18n

### Lovable Prompt
```
Add CSS custom properties to src/index.css inside :root (keep existing styles):

--color-primary: #1E6B3F;
--color-primary-dark: #155130;
--color-primary-light: #EBF5EE;
--color-accent: #F4A11B;
--color-accent-light: #FDF6E3;
--color-surface: #FFFFFF;
--color-surface-alt: #F9FAFB;
--color-border: #E5E7EB;
--color-text-primary: #111827;
--color-text-secondary: #6B7280;
--color-text-muted: #9CA3AF;
--color-success: #059669;
--color-warning: #D97706;
--color-error: #DC2626;
--color-bourse-bg: #111827;
--color-bourse-card: #1F2937;
--color-bourse-accent: #F59E0B;
--phone-width: 393px;
--status-bar-height: 44px;
--bottom-nav-height: 80px;

Add to tailwind.config.ts colors:
'mombongo-green': '#1E6B3F'
'mombongo-amber': '#F4A11B'

Add to index.html <head> (before </head>):
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700&family=Noto+Sans:wght@400;500;600&display=swap" rel="stylesheet">

Create src/lib/i18n.ts:
import i18n from 'i18next'
import { initReactI18next } from 'react-i18next'
import LanguageDetector from 'i18next-browser-languagedetector'
import fr from '@/locales/fr.json'
import en from '@/locales/en.json'
import ln from '@/locales/ln.json'

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: { fr: { translation: fr }, en: { translation: en }, ln: { translation: ln } },
    fallbackLng: 'fr',
    supportedLngs: ['fr', 'en', 'ln'],
    detection: {
      order: ['localStorage'],
      caches: ['localStorage'],
      lookupLocalStorage: 'mombongo-language',
    },
    interpolation: { escapeValue: false },
  })

export default i18n

Create src/locales/fr.json:
{
  "nav": {
    "market": "Marché", "bourse": "Bourse", "financing": "Financement",
    "academia": "Academia", "profile": "Profil"
  },
  "common": {
    "invest": "Investir", "loading": "Chargement...",
    "error": "Une erreur s'est produite", "retry": "Réessayer",
    "save": "Enregistrer", "cancel": "Annuler", "confirm": "Confirmer",
    "back": "Retour", "seeAll": "Voir tout",
    "offline": "Pas de connexion. Certaines données peuvent être obsolètes.",
    "search": "Rechercher"
  },
  "auth": {
    "login": "Se connecter", "signup": "S'inscrire",
    "email": "Adresse email", "password": "Mot de passe",
    "fullName": "Nom complet", "role": "Je suis...",
    "forgotPassword": "Mot de passe oublié?",
    "roleInvestor": "Investisseur", "roleFarmer": "Agriculteur",
    "roleMerchant": "Commerçant",
    "error": {
      "userNotFound": "Aucun compte avec cet email",
      "wrongPassword": "Mot de passe incorrect",
      "emailInUse": "Email déjà utilisé",
      "weakPassword": "Mot de passe trop court (6 caractères min.)",
      "network": "Pas de connexion. Vérifiez votre connexion."
    }
  },
  "market": {
    "searchPlaceholder": "Chercher un produit...",
    "noResults": "Aucun résultat", "filter": "Filtrer",
    "roi": "Rendement", "duration": "Durée", "minInvest": "Min. invest.",
    "investors": "investisseurs", "funded": "Financé",
    "featured": "En vedette"
  }
}

Create src/locales/en.json with same keys in English.
Create src/locales/ln.json with same keys in Lingala.

Create src/lib/utils.ts (add to existing file, don't overwrite):
export function formatUsd(amount: number): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount)
}
export function formatCdf(amount: number): string {
  return new Intl.NumberFormat('fr-CD', { maximumFractionDigits: 0 }).format(amount) + ' FC'
}
export function formatPercent(value: number): string {
  return `${value.toFixed(1)}%`
}
```

### Regression
```bash
bun run typecheck && bun run lint
```

---

## Step 2 — AppShell + Layout Components

### Lovable Prompt
```
Create these layout components. Do not modify existing Lovable components.

src/components/layout/StatusBar.tsx:
Height 44px. Left: current time (HH:MM, updates every 60s via setInterval).
Right: Wifi icon + Battery icon (lucide-react) + "98%".
Props: { style?: 'light' | 'dark' } — controls text/icon color.
Transparent background.

src/components/layout/AppHeader.tsx:
Height 56px. px-4. bg-white. border-b border-gray-100.
Props: { title?: string; showBack?: boolean; onBack?: () => void;
         showNotification?: boolean; showLanguage?: boolean }
Left: ChevronLeft (calls onBack) if showBack, else "🌿 Mombongo" in text-[#1E6B3F] font-bold.
Center: title string.
Right: language pill (shows FR/EN/LN) + Bell icon with red badge if showNotification.
Language pill opens a shadcn Sheet from bottom with 3 options:
  🇫🇷 Français | 🇬🇧 English | 🇨🇩 Lingala
  On select: i18n.changeLanguage(lang), localStorage.setItem('mombongo-language', lang), close.

src/components/layout/BottomNav.tsx:
Height 80px. bg-white. border-t border-gray-100.
5 tabs using NavLink from react-router-dom:
  ShoppingBag icon → /market → t('nav.market')
  BarChart3 icon → /bourse → t('nav.bourse')
  Sprout icon → /financing → t('nav.financing')
  GraduationCap icon → /academia → t('nav.academia')
  User icon → /profile → t('nav.profile')
Active tab: text-[#1E6B3F]. Inactive: text-gray-400.
Label below icon, font-size 10px.
Add data-testid="bottom-nav-{route}" to each tab link.

src/components/layout/AppShell.tsx:
Props: { children: ReactNode; hideBottomNav?: boolean }
On desktop (>768px): centers content in a phone frame:
  width: var(--phone-width) = 393px
  min-height: 852px
  border-radius: 48px
  box-shadow: 0 25px 60px rgba(0,0,0,0.25)
  overflow: hidden
  Outer background: bg-gray-100
On mobile (≤768px): full width, no frame.
Use CSS media query for the toggle.
Structure: <StatusBar /> + <main> (overflow-y-auto, padding-bottom 80px) + <BottomNav /> (if !hideBottomNav)

src/components/ui/SkeletonLoader.tsx:
Props: { variant?: 'card' | 'list' | 'text'; lines?: number; className?: string }
Shimmer animation: CSS keyframes sliding gradient.
card: rounded-xl h-48 w-full.
list: stacked rows varying width.
text: single line full width.
Add role="status" aria-label="Loading".
```

### Regression
```bash
bun run test:unit
bun run build
```

📝 Manual: `bun run dev` → open at 1280px width → phone frame visible.

---

## ✅ Milestone — S0-03 Complete
- [ ] CSS variables in `:root`
- [ ] Google Fonts loading (check Network tab)
- [ ] i18n: `t('nav.market')` returns 'Marché' in FR
- [ ] All 3 locale files have identical keys
- [ ] AppShell renders phone frame on desktop
- [ ] BottomNav has 5 tabs with correct routes

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0
- [ ] `bun run build` exits 0
- [ ] Zero hardcoded translatable strings — all use `t()`

```bash
git add -A
git commit -m "feat(s0-03): design tokens, i18n FR/EN/LN, AppShell, AppHeader, BottomNav"
git push origin feature/s0-03-design-system
```
