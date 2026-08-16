# SGN-06 — App Store Submission: Google Play + Apple App Store

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SGN-06 |
| Repos | `mombongo-mobile` |
| Branch | `feature/sgn-06-store-submission` |
| Merges into | `main` (release branch) |
| Priority | P1 — Mobile MVP launch gate |
| Estimate | 8h (build prep 4h + store listings 4h) |

## Why this matters
The mobile app is built with Expo/EAS. To reach users on their phones, we need to publish to
Google Play (Android — primary target for DRC) and eventually the Apple App Store (iOS diaspora users).
Google Play is critical: ~95% of DRC smartphone users are on Android.

---

## Pre-requisites (check before starting)
- [ ] `eas.json` configured with `production` profile
- [ ] Android: Google Play Developer account created (one-time $25 fee)
- [ ] iOS: Apple Developer Program enrolled ($99/year) — optional for DRC MVP
- [ ] All SM-00 through SM-10 sprints complete (or at minimum SM-00, SM-04, SM-05, SM-09)
- [ ] App has been smoke-tested on a real Android device
- [ ] Privacy Policy URL live (SGN-02)
- [ ] Support email set: support@mombongo.com

---

## Work items

### 1. App metadata & assets

**`app.json` / `app.config.js` updates**:
```json
{
  "name": "Mombongo",
  "slug": "mombongo",
  "version": "1.0.0",
  "description": "Finance agricole — investissement, crédit et marché pour l'Afrique",
  "android": {
    "package": "com.mombongo.app",
    "versionCode": 1,
    "permissions": ["CAMERA", "READ_EXTERNAL_STORAGE", "NOTIFICATIONS", "INTERNET", "VIBRATE"]
  },
  "ios": {
    "bundleIdentifier": "com.mombongo.app",
    "buildNumber": "1"
  }
}
```

**Store assets to create**:
- App icon: 1024×1024 PNG (no alpha), dark green with "M" logo
- Adaptive icon (Android): foreground 108×108dp + background color
- Splash screen: 1284×2778 (iPhone 14 Pro Max), matching brand colors
- Feature graphic (Google Play): 1024×500 PNG
- Screenshots: minimum 4, for 6.5" phone
  - Screen 1: Home screen (investor dashboard)
  - Screen 2: Market / product listings
  - Screen 3: Invest flow (PaymentModal step 1)
  - Screen 4: Farmer exploitation screen

### 2. Google Play submission

**Build**:
```bash
# In mombongo-mobile:
eas build --platform android --profile production
# This generates a signed AAB (Android App Bundle)
```

**Google Play Console steps**:
1. Create new app → "Mombongo" → French (primary) + English
2. Set up app signing (Google-managed or upload key)
3. Content rating: "Finance" category → complete questionnaire
4. Target audience: Adults 18+ (financial app)
5. Privacy Policy URL: `https://mombongo-web.pages.dev/confidentialite`
6. Store listing:
   - Short description (80 chars): "Investissement, crédit agricole et marché en DRC"
   - Full description (4000 chars): see copy below
7. Upload AAB to Internal Testing track → promote to Production after review

**App store description (French)**:
```
Mombongo est la plateforme de finance agricole pour la République Démocratique du Congo.

🌿 POUR LES AGRICULTEURS
• Vendez vos récoltes directement aux commerçants
• Demandez un financement agricole en quelques minutes
• Suivez votre exploitation : cultures, récoltes, revenus

💰 POUR LES INVESTISSEURS
• Investissez dans des projets agricoles vérifiés à partir de 50 USD
• Déposez via Airtel Money, Orange Money, M-Pesa ou carte bancaire
• Suivez vos rendements en temps réel

🤝 POUR LES COMMERÇANTS
• Achetez directement auprès de milliers d'agriculteurs en DRC
• Négociez les prix et signez des contrats numériques
• Paiement sécurisé par escrow

✅ SÉCURISÉ ET FIABLE
• Vérification d'identité (KYC) obligatoire
• Paiements via opérateurs mobile money certifiés
• Données chiffrées et protégées

Disponible en français. Support WhatsApp disponible.
```

### 3. Apple App Store submission (optional for DRC MVP, required for diaspora)

**Build**:
```bash
eas build --platform ios --profile production
```

**App Store Connect steps**:
1. Create new app in App Store Connect
2. Category: Finance
3. Age Rating: 17+ (financial transactions)
4. App Review Information: provide test account credentials
5. Privacy labels: Financial Info, User Content, Contact Info, Identifiers
6. Submit for review (7–14 day review typical for Finance category)

### 4. EAS Update (OTA updates after submission)
Configure EAS Update for over-the-air JS updates:
```bash
eas update:configure
# This allows pushing JS-only fixes without full store review
```

```json
// eas.json — add update channel:
{
  "build": {
    "production": {
      "channel": "production"
    }
  }
}
```

### 5. Post-launch monitoring checklist
- [ ] Firebase Crashlytics alerts configured
- [ ] Play Store rating reply template prepared in French
- [ ] Support WhatsApp number (from SGN-03) staffed and monitored
- [ ] Analytics events verified firing (from SM-10)

---

## Acceptance criteria
- [ ] `eas build --platform android --profile production` exits 0
- [ ] AAB uploaded to Google Play Internal Testing
- [ ] Store listing complete in French with all required screenshots
- [ ] Privacy Policy URL live and accessible
- [ ] App installable from Play Store on a DRC test device
- [ ] EAS Update configured for OTA JS fixes

---

## Implementation Status
NOT DONE
