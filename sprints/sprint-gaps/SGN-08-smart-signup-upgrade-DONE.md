# SGN-08 — Smart Signup Upgrade & Role-Specific Onboarding Form

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-08 |
| Repos | `mombongo-web`, `mombongo-mobile`, `mombongo-functions` |
| Branch | `feature/sgn-08-smart-signup` |
| Merges into | `dev` |
| Priority | P1 — Non-tech users pick the wrong role and get stuck |
| Estimate | 4h |
| Depends on | SGN-02 (T&C checkbox) |

## Why this matters
SG-01 spec already covers the full vision. This sprint implements the most critical parts for MVP:
simplified role selection + role-specific quick fields + agent invite-only path.
The current signup gives all 4 roles equal billing with no guidance — a farmer will not know
what "investisseur" or "commerçant" means without context.

---

## Work items (subset of SG-01 for MVP)

### 1. Guided role selection screen
Replace the 4-card grid with 3 options + guided language:
- "J'investis dans l'agriculture" → investor
- "Je suis agriculteur" → farmer
- "J'achète des produits agricoles" → merchant
- Agent is NOT self-selectable — hidden/invite-only
  - Small text at bottom: "Vous êtes agent terrain ? [Contactez-nous]" → WhatsApp link

Each option:
- Large icon (🌿 / 💰 / 🚚)
- Short description in plain French (max 10 words)
- Highlighted "Choisir" button on tap

### 2. Role-specific quick fields (step between role select and account creation)
Show 2–3 role-relevant fields before email/password (to personalize the experience):

**Farmer**: Province (dropdown: Kinshasa, Katanga, Kongo-Central, Nord-Kivu, Sud-Kivu, Kasaï, Autre) + Main crop (dropdown)
**Investor**: Country (DRC / Diaspora toggle) + Phone number
**Merchant**: Business type (Transporteur / Grossiste / Transformateur / Exportateur)

These fields are stored to `users/{uid}` doc by the `createUserProfile` CF.
Update CF to accept the new fields.

### 3. Phone number collection at signup
Currently no phone number is collected. Phone is required for mobile money (PawaPay).
Add phone field to signup step 2. Format: +243 XXX XXX XXX.
Basic validation: must start with +243 (DRC) or allow international format.
Store to `users/{uid}.phone`.

### 4. Signup confirmation screen
After account creation, instead of immediately landing on Home:
- Show a "Bienvenue, [Prénom] ! 🎉" screen
- 3 bullets: "Votre compte est créé" / "Vérifiez votre email" / "Complétez votre profil (KYC)"
- "Commencer" → navigates to onboarding (SGN-01) or directly to Home

### 5. Google Sign-In role assignment
Currently Google Sign-In assigns no role → user lands with `role: undefined`.
After Google Sign-In, if `users/{uid}` doc doesn't exist or has no role:
- Show the role selection screen before proceeding to Home
- Same flow as manual signup step 1 (guided role selection)

---

## Cloud Functions changes
- `createUserProfile` CF: add `phone`, `province`, `cropType`, `businessType`, `country` optional fields

---

## Acceptance criteria
- [ ] Role selection shows 3 options with plain French guidance (no "investor/farmer/merchant" jargon)
- [ ] Agent role is not self-selectable
- [ ] Role-specific quick fields shown between role select and email/password
- [ ] Phone number collected and stored at signup
- [ ] Google Sign-In triggers role selection if no role set
- [ ] Post-signup confirmation screen shown
- [ ] `npx tsc --noEmit` passes

---

## Implementation Status
NOT DONE
