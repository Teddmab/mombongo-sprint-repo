# SGN-07 — Accessibility, CDF Display & Localization for DRC Users

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-07 |
| Repos | `mombongo-web`, `mombongo-mobile` |
| Branch | `feature/sgn-07-accessibility-cdf` |
| Merges into | `dev` |
| Priority | P1 — Non-tech-savvy DRC users may struggle with USD-only display, small text, or poor contrast |
| Estimate | 4h |

## Why this matters
Target users include farmers in rural DRC who:
- Think in Congolese Francs (CDF), not USD
- May have low vision or use older low-resolution screens
- May not be fluent in "financial French" (ROI, escrow, KYC are jargon)
- May share their phone with family members

This sprint makes the app legible, accessible, and culturally grounded.

---

## Work items

### 1. CDF / USD dual display

**Context**: 1 USD ≈ 2,800 CDF (rate fluctuates). Store a reference rate in Firestore (`config/exchangeRate`).

**New CF**: `getExchangeRate()` — returns current USD/CDF rate from `config/exchangeRate` doc.

Update `formatUsd` utility to also return CDF equivalent:
```typescript
// src/lib/utils.ts
export function formatMoney(usd: number, rate: number): string {
  const cdf = Math.round(usd * rate)
  return `$${usd.toFixed(2)} (${cdf.toLocaleString('fr-FR')} CDF)`
}
```

Show CDF on:
- Wallet balance (primary: USD, secondary: CDF in smaller text)
- Investment amounts
- Market listing prices
- Financing amounts
- Deposit / withdraw amounts

Example display:
```
Portefeuille: $245.00
              ≈ 686 000 CDF
```

**Rate update**: Admin can update the rate in `mombongo-admin` settings. Rate cached in TanStack Query for 1h.

### 2. Terminology glossary (tooltip on jargon)
Replace or supplement jargon with plain French:
| Technical term | Plain French shown to user |
|----------------|---------------------------|
| KYC | Vérification d'identité |
| ROI | Rendement (ex: +22%) |
| Escrow | Paiement sécurisé |
| Wallet | Mon portefeuille |
| Portfolio | Mes investissements |
| Bourse | Marché agricole |
| Tranche | Versement partiel |

Add a `?` icon next to any remaining technical terms that opens a 1-sentence tooltip.

### 3. Font size and readability
- Body text minimum: 14px (currently OK but verify)
- Important numbers (balances, amounts): 18–24px, bold
- Line height: 1.5 for all body text
- Ensure sufficient color contrast (WCAG AA: 4.5:1 for normal text, 3:1 for large text)

Check and fix contrast issues on:
- Muted text on dark backgrounds
- Pill/badge text
- Form placeholder text

### 4. Large touch targets
All tappable elements: minimum 44×44px touch area (WCAG 2.1).
Check:
- Nav bar icons
- Close buttons on modals
- Row items in lists (they should have `min-height: 48px`)

### 5. Error messages in plain French
Audit all error states and replace technical Firebase errors with user-friendly French:
```typescript
// src/lib/errors.ts — utility to map Firebase error codes
export function friendlyError(code: string): string {
  const map: Record<string, string> = {
    'auth/wrong-password': 'Mot de passe incorrect. Vérifiez et réessayez.',
    'auth/user-not-found': 'Aucun compte avec cet email. Avez-vous un compte ?',
    'auth/too-many-requests': 'Trop de tentatives. Attendez quelques minutes et réessayez.',
    'auth/network-request-failed': 'Pas de connexion internet. Vérifiez votre réseau.',
    'functions/unavailable': 'Service temporairement indisponible. Réessayez dans un instant.',
    'functions/unauthenticated': 'Session expirée. Reconnectez-vous.',
  }
  return map[code] ?? 'Une erreur est survenue. Contactez le support si le problème persiste.'
}
```

### 6. Offline indicator
Show a persistent banner when the device has no connectivity:
```tsx
// src/components/OfflineBanner.tsx
// Uses navigator.onLine + window 'offline'/'online' events (web)
// Uses NetInfo from @react-native-community/netinfo (mobile)
// Banner: "Hors ligne — certaines fonctionnalités ne sont pas disponibles"
// Disappears automatically when connection is restored
```

### 7. i18n preparation (Lingala / Swahili — future-ready)
The app already uses `react-i18next`. Ensure all strings are in translation keys, not hardcoded.
Audit and extract any remaining hardcoded French strings to `src/i18n/fr.json`.
(Lingala/Swahili translation is out of scope for this sprint, but the strings must be extractable.)

---

## Cloud Functions
- `getExchangeRate()` — returns `{ rate: number, updatedAt: Timestamp }`
- Admin: update rate via a settings page (write to `config/exchangeRate` doc)

---

## Acceptance criteria
- [ ] Wallet balance shows USD + CDF equivalent everywhere
- [ ] No hardcoded jargon without a plain-language label or tooltip
- [ ] All touch targets ≥ 44×44px
- [ ] All error messages in friendly French (no Firebase error codes visible to user)
- [ ] Offline banner appears and disappears correctly
- [ ] WCAG AA contrast passes for all text (verify with browser DevTools)
- [ ] `npx tsc --noEmit` passes

---

## Implementation Status
NOT DONE
