# SGN-01 — Onboarding Tutorial: First-Time User Experience

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-01 |
| Repos | `mombongo-web`, `mombongo-mobile` |
| Branch | `feature/sgn-01-onboarding` |
| Merges into | `dev` |
| Priority | P1 — MVP required (non-tech-savvy users will drop off without guidance) |
| Estimate | 5h (web 3h + mobile 2h) |

## Why this matters
First-time users land on their role's Home screen with zero explanation of what to do next.
For a farmer in Goma who has never used a finance app, this is a wall. We need a short, illustrated,
skippable welcome sequence that tells them what Mombongo does for them and what to do first.
No jargon. No walls of text. Maximum 4 screens.

## Design principle
- One idea per screen
- Large icon/illustration + short title + 1–2 sentences max
- CTA: "Suivant" or "Commencer" (never "Next" or "Skip" as first option)
- Skip button small and secondary in the top-right corner

---

## Work items

### 1. `OnboardingModal` component (web)
**File**: `src/components/OnboardingModal.tsx`

Show this modal once, immediately after first login (when `localStorage.getItem('onboarding_done')` is null).

Role-specific slides (4 slides each):

**Farmer slides**:
1. "👋 Bienvenue sur Mombongo" — Votre assistant agricole et financier personnel.
2. "📦 Vendez vos récoltes" — Publiez vos produits sur le marché. Les commerçants vous contactent directement.
3. "💰 Obtenez un financement" — Demandez un crédit agricole et suivez votre dossier depuis l'app.
4. "📊 Suivez votre exploitation" — Gérez vos cultures, vos récoltes et vos revenus en un seul endroit.

**Investor slides**:
1. "👋 Bienvenue sur Mombongo" — Investissez dans l'agriculture africaine. Simple, sécurisé, rentable.
2. "🌿 Choisissez un produit" — Parcourez les projets agricoles vérifiés. À partir de 50 USD.
3. "💳 Déposez des fonds" — Via Airtel Money, Orange Money, M-Pesa ou carte bancaire.
4. "📈 Suivez vos rendements" — Votre tableau de bord montre vos gains et l'avancement de chaque projet.

**Agent terrain slides**:
1. "👋 Bienvenue sur Mombongo" — Vous êtes le lien entre les agriculteurs et les financeurs.
2. "🗺️ Vos agriculteurs" — Consultez la liste des agriculteurs qui vous sont assignés.
3. "📝 Rapports de terrain" — Soumettez vos rapports avec photos pour valider les tranches.
4. "✅ Validez les jalons" — Chaque rapport approuvé débloque un versement pour l'agriculteur.

**Merchant slides**:
1. "👋 Bienvenue sur Mombongo" — Achetez des produits agricoles directement auprès des producteurs.
2. "🔍 Parcourez le marché" — Consultez les offres de milliers d'agriculteurs en DRC.
3. "📋 Passez des commandes" — Proposez un prix, négociez, et signez un contrat numérique.
4. "🚚 Suivez vos livraisons" — Confirmez la réception et libérez le paiement via l'escrow Mombongo.

**Implementation**:
```tsx
// LocalStorage key: 'mb_onboarding_done'
// Show if key doesn't exist in localStorage after auth
// Set key on last slide "Commencer" click
// Also accessible from ProfileScreen as "Revoir le tutoriel"
```

### 2. Role-specific tips banner (persistent, dismissible)
After onboarding, show a small tip banner on Home for 7 days:
- Farmer: "Astuce : publiez votre première annonce sur le Marché pour être visible des commerçants."
- Investor: "Astuce : vérifiez votre identité pour pouvoir déposer et retirer des fonds."
- Agent: "Astuce : consultez votre liste d'agriculteurs pour soumettre votre premier rapport."
- Merchant: "Astuce : explorez le marché pour trouver des produits frais près de chez vous."

**Implementation**:
```tsx
// LocalStorage key: 'mb_tip_dismissed_<role>'
// Show dismissible banner with ✕ button
// Auto-dismiss after 7 days (store timestamp)
```

### 3. Mobile (mombongo-mobile) — `OnboardingScreen`
Same flow, but full-screen slides with large illustrations and swipe gesture.
- Use `FlatList` with `pagingEnabled` or a dedicated slide library
- Dot indicator at bottom
- "Passer" button top-right
- "Commencer" button on final slide → navigates to Home and sets `AsyncStorage` key

---

## Cloud Functions needed
None — purely frontend with localStorage/AsyncStorage.

---

## Acceptance criteria
- [ ] On first login (web), modal appears with role-specific slides
- [ ] "Commencer" sets localStorage key — modal never shows again
- [ ] "Passer" also sets key — dismisses without completing
- [ ] Accessible from Profile: "Revoir le tutoriel"
- [ ] Mobile: full-screen onboarding with swipe gesture on first launch
- [ ] All text in French, no English or jargon visible
- [ ] `npx tsc --noEmit` passes
- [ ] `npx vitest run` passes

---

## Implementation Status
NOT DONE
