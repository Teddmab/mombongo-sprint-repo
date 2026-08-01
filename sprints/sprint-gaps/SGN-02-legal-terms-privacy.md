# SGN-02 — Legal: Terms of Service + Privacy Policy Pages

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-02 |
| Repos | `mombongo-web` |
| Branch | `feature/sgn-02-legal-pages` |
| Merges into | `dev` |
| Priority | P0 — Launch blocker (financial apps require T&C acceptance by law) |
| Estimate | 2h |

## Why this matters
Mombongo handles real money (deposits, investments, mobile money). Without Terms of Service and a
Privacy Policy, the app cannot legally onboard users in DRC or for diaspora users in the EU/US.
Firebase and PawaPay also require these for payment processing approval.

---

## Work items

### 1. Legal pages (static routes)
Create two static pages:
- `/mentions-legales` — Terms of Service
- `/confidentialite` — Privacy Policy

Both pages:
- Simple, readable layout (not a wall of text — use sections with headers)
- Language: French
- Footer links to both pages (from every page in the app)
- Last updated date visible at top

**Terms of Service key sections**:
1. Objet de la plateforme
2. Inscription et compte utilisateur
3. Investissement et risques (disclaimer: agriculture involves risks, no guaranteed returns)
4. Dépôts, retraits et frais (5% platform fee disclosed)
5. Rôles et responsabilités (investisseur / agriculteur / agent / commerçant)
6. Contenu interdit
7. Résiliation de compte
8. Loi applicable (DRC / OHADA)
9. Coordonnées de contact

**Privacy Policy key sections**:
1. Données collectées (nom, email, téléphone, KYC documents, transactions)
2. Utilisation des données
3. Conservation et sécurité
4. Partage avec des tiers (PawaPay, Stripe, Firebase)
5. Droits de l'utilisateur (accès, suppression, portabilité)
6. Cookies et Analytics
7. Contact: privacy@mombongo.com

### 2. Acceptance checkbox at signup
On AuthScreen signup step (before account creation):
- Add checkbox: "J'accepte les [Conditions d'utilisation] et la [Politique de confidentialité]"
- Links open the pages in a new tab
- "Créer mon compte" button is disabled until checkbox is checked
- Store `termsAcceptedAt: serverTimestamp()` on `users/{uid}` doc via `createUserProfile` CF

### 3. Footer component
Add a minimal footer to the web app visible on auth and landing pages:
```tsx
// src/components/Footer.tsx
// Links: Conditions d'utilisation · Politique de confidentialité · Contact
// Shown on AuthScreen, below the main navigation
```

### 4. Legal update banner (future-proof)
If `termsAcceptedAt` is before the last terms update date, show a banner:
"Nos conditions ont été mises à jour. Veuillez les relire et accepter pour continuer."
(For now, store last update date in a const — no CF needed)

---

## Cloud Functions changes
- `createUserProfile` CF: add `termsAcceptedAt` field to the user document

---

## Acceptance criteria
- [ ] `/mentions-legales` page renders with all sections in French
- [ ] `/confidentialite` page renders with all sections in French
- [ ] Signup has T&C checkbox — button disabled without it
- [ ] `termsAcceptedAt` stored on user doc in Firestore
- [ ] Footer links visible on AuthScreen
- [ ] `npx tsc --noEmit` passes

---

## Implementation Status
NOT DONE
