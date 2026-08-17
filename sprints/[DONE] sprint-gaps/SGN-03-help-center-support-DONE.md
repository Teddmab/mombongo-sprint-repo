# SGN-03 — Help Center & In-App Support

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-03 |
| Repos | `mombongo-web`, `mombongo-mobile` |
| Branch | `feature/sgn-03-help-center` |
| Merges into | `dev` |
| Priority | P1 — Non-tech users need help access without leaving the app |
| Estimate | 3h |

## Why this matters
The target users — farmers, small merchants in DRC — are not used to financial apps. When something
goes wrong or they don't understand a feature, they need immediate help in French. Without a help
entry point, they abandon the app. A WhatsApp support button is the culturally correct solution for DRC.

---

## Work items

### 1. Help screen (`/aide`, `src/pages/HelpScreen.tsx`)
Accessible from: Profile screen → "Aide & Support" menu item.

Structure:
```
[Search bar — filters FAQ items]

FOIRE AUX QUESTIONS

📱 Mon compte
  - Comment créer un compte ? [expandable]
  - Comment changer mon mot de passe ? [expandable]
  - Comment vérifier mon identité (KYC) ? [expandable]
  - Comment modifier mon profil ? [expandable]

💰 Paiements et dépôts
  - Comment déposer des fonds ? [expandable]
  - Quels opérateurs mobile money acceptez-vous ? [expandable]
  - Combien de temps prend un dépôt ? [expandable]
  - Comment retirer mes fonds ? [expandable]
  - Quels sont les frais de la plateforme ? (5% sur les transactions) [expandable]

🌿 Investissements
  - Comment choisir un produit agricole ? [expandable]
  - Quand vais-je recevoir mes rendements ? [expandable]
  - Que se passe-t-il si la récolte échoue ? [expandable]
  - Puis-je retirer mon investissement avant terme ? [expandable]

📦 Marché (Agriculteurs)
  - Comment publier une annonce ? [expandable]
  - Comment modifier une annonce ? [expandable]
  - Comment un commerçant me contacte-t-il ? [expandable]

📋 Financement (Agriculteurs)
  - Comment demander un financement ? [expandable]
  - Qui est mon agent terrain ? [expandable]
  - Comment se passent les versements ? [expandable]
```

Each FAQ item:
- Title (clickable, expands/collapses)
- Answer: max 3 sentences in plain French
- Optional "Contacter le support" link at bottom of answer

### 2. WhatsApp Support button (floating or in Help screen)
```tsx
// Opens wa.me/+243XXXXXXXXX with a pre-filled message:
// "Bonjour Mombongo, j'ai besoin d'aide avec [mon compte / un paiement / ...]"
// Replace +243XXXXXXXXX with real Mombongo support number
const SUPPORT_WHATSAPP = "+243XXXXXXXXX" // TODO: replace with real number
const SUPPORT_MESSAGE = encodeURIComponent("Bonjour Mombongo, j'ai besoin d'aide.")
const url = `https://wa.me/${SUPPORT_WHATSAPP}?text=${SUPPORT_MESSAGE}`
```

Button placement:
- Help screen: large green "💬 Contacter le support sur WhatsApp" button at bottom
- ProfileScreen: "Aide & Support" → navigates to `/aide`
- On every error screen / toast: "Besoin d'aide ? Contactez-nous sur WhatsApp"

### 3. Contextual help tooltips
On key screens that might confuse users, add a `?` icon that shows a brief tooltip:
- Wallet screen: "Qu'est-ce qu'un dépôt ?"
- Investment screen: "Qu'est-ce que le ROI ?"
- KYC screen: "Pourquoi avons-nous besoin de ces documents ?"
- Bourse screen (farmer): "À quoi sert la Bourse ?"

### 4. Mobile (mombongo-mobile) — `HelpScreen`
Same FAQ structure but as an accordion `FlatList`.
WhatsApp button opens via `Linking.openURL('https://wa.me/...')`.

---

## Cloud Functions needed
None — static content + external link.

---

## Acceptance criteria
- [ ] `/aide` page renders with searchable FAQ accordion
- [ ] WhatsApp support button opens wa.me link with pre-filled message
- [ ] All content in French, no jargon
- [ ] ProfileScreen links to Help screen
- [ ] `?` tooltips on at least 4 key screens
- [ ] Mobile help screen accessible from mobile profile tab
- [ ] `npx tsc --noEmit` passes

---

## Implementation Status
NOT DONE
