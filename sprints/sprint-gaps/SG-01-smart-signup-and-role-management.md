# SG-01 — Smart Signup & Role Management

## Why this matters
Right now every role sees the exact same signup form. A farmer is not a tech-savvy investor. If someone picks the wrong role, they're stuck — there is no self-service fix. The goal here is to make signup feel like a conversation, not a form.

## Current problems
- All 4 roles shown as equal options at signup — no guidance on which to pick
- Signup step 2 collects only name + email + password regardless of role
- Wrong role = stuck forever (must contact admin)
- Google Sign-in assigns no role (user lands on home with undefined role)
- No onboarding after signup — user lands directly on Home screen with zero context

## Work items

### 1. Smarter role selection screen
- Replace the 4-card grid with a single guided question: "Comment allez-vous utiliser Mombongo ?"
- Show 3 simplified choices (merge agent into a separate invite-only path):
  - **"J'investis dans l'agriculture"** → investor
  - **"Je suis agriculteur"** → farmer
  - **"Je suis commerçant"** → merchant
- Agent is NOT self-selectable — agents are invited/created by admins only (role = invite-only)
- Each choice has a 1-line description written in plain language, not jargon

### 2. Role-specific signup step 2
After role selection, collect role-relevant info before account creation:

**Farmer**:
- Phone number (required — used for mobile money)
- Province / territory
- Primary crop (dropdown: maïs, manioc, riz, haricot, autre)
- Approximate surface (ha) — optional, simplified: "moins de 1 ha / 1–5 ha / plus de 5 ha"

**Investor**:
- Phone number
- Country of residence (DRC or diaspora)
- How they heard about Mombongo (optional)

**Merchant**:
- Phone number
- Business type (transporteur, grossiste, transformateur, exportateur)
- Province of operation

### 3. Onboarding screen after first signup
- One-time "Bienvenue" screen after account creation (check `isFirstLogin` flag in Firestore)
- 3 slides max, no more: "Voici comment ça marche"
- Role-specific: farmer sees financing steps, investor sees investment steps, merchant sees bourse steps
- "C'est parti !" CTA navigates to Home and sets `isFirstLogin: false`

### 4. Google Sign-in role assignment
- If Google sign-in and no existing profile: redirect to role selection before going to home
- After role is selected, call `createUserProfile` CF with the chosen role
- Never skip role assignment

### 5. Self-service role change request
- Settings screen → "Mon rôle" → "Demander un changement de rôle"
- Simple form: current role (read-only), requested role, reason (text field)
- Creates a doc in `role_change_requests` collection
- Admin sees these in AdminUsers and can approve (calls `setUserRole` CF)
- User gets a push notification when approved

### 6. Agent is invite-only
- Remove "Agent terrain" from the public signup role list
- Admins create agent accounts from AdminUsers screen (email invite or manual create)
- Agent receives email with a magic link or temporary password

## Acceptance criteria
- [ ] Signup shows 3 choices (investor / farmer / merchant), not 4
- [ ] Each role collects relevant info at signup (step 2 adapts)
- [ ] Onboarding slides shown once after first login
- [ ] Google Sign-in prompts for role selection if no profile exists
- [ ] Role change request form exists in Settings → submitted to Firestore
- [ ] Agent role is not selectable at signup
