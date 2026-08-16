# SF-07 — Agronomy Advisory & Province Weather Push Alerts

## Goal
Province-level weather warnings (floods, drought, pest outbreaks) matched to farmer's registered crops and current growth stage. Advisories pushed via FCM; an AgroAdvisoryCard on AgricultorHome shows the 1–2 most critical. A scheduled CF sends crop-stage tips (e.g. "Maïs stade floraison — appliquez engrais azoté cette semaine"). Helps farmers take protective action before crop damage occurs.

## Status
**TODO** · Priority P2 · Est. 4h · Web

## Branch
`feature/sf-07-agro-advisory`

## Repos
- `mombongo-web`
- `mombongo-functions`

---

## Firestore

### Collection: `agro_advisories`
```
agro_advisories/{advisoryId}
  province: string               // "Kinshasa" | "Kongo-Central" | ... | "all" for national
  commodity?: string             // optional — null means all commodities
  growthStage?: string           // optional — matches cultural calendar stages
  title: string                  // e.g. "⚠️ Alerte sécheresse — Kasaï Oriental"
  body: string                   // full advisory text
  severity: 'info' | 'warning' | 'critical'
  source: string                 // e.g. "Direction Nationale Météo · METTELSAT"
  effectiveFrom: Timestamp
  effectiveTo: Timestamp
  createdBy: string              // admin uid
  createdAt: Timestamp
```

**Who writes:** Agronomist/admin team via `mombongo-admin` simple form (or direct Firestore console until admin panel is built).

---

## Cloud Functions

### `getAgroAdvisory` (callable)
**Input:**
```typescript
{
  province: string               // caller's province
  commodities: string[]          // from useMyCultures() — farmer's registered crops
}
```
**Logic:**
1. Query `agro_advisories` where:
   - `(province === caller.province || province === 'all')`
   - `(commodity === null || commodity in commodities)`
   - `effectiveFrom <= now <= effectiveTo`
2. Order by `severity` (critical first), then `effectiveFrom` desc
3. Return top 5

---

### `subscribeToProvinceAlerts` (callable)
**Input:**
```typescript
{
  province: string
  commodities: string[]
}
```
**Logic:** Updates `users/{uid}.advisorySubscriptions = { province, commodities, subscribedAt }`. Used by `sendCropStageAlerts` to know which users to notify.

---

### `sendCropStageAlerts` (scheduled — daily 07:00 DRC time)
**Logic:**
1. Fetch all `cultures` where today's date falls within the growth stage transition window (based on `startDate + stage durations` from cultural calendar)
2. For each culture entering a new stage today:
   - Look up stage-specific agronomy tips from a hardcoded `CROP_STAGE_TIPS` map (see below)
   - Send FCM to the culture's `farmerId`
3. `CROP_STAGE_TIPS` examples:
   ```
   Manioc/Plantation      → "Plantez les boutures à 1m × 1m d'espacement"
   Manioc/Croissance      → "Désherbez autour des plants — évitez compétition racines"
   Maïs/Floraison         → "Stade critique: appliquez engrais azoté (urée) maintenant"
   Maïs/Maturation        → "Arrêtez irrigation — laissez sécher sur pied 2 semaines"
   Café/Récolte           → "Récoltez cerises rouges uniquement — tri à 100%"
   Cacao/Post-récolte     → "Fermentation 5–7 jours sous sacs jute, retournez 2×/jour"
   ```

---

## Frontend

### Hook: `src/hooks/useAgroAdvisory.ts`
```typescript
export function useAgroAdvisory(): AgroAdvisory[]
export function useSubscribeToProvinceAlerts(): UseMutationResult<...>
```

Mock:
```typescript
const MOCK_ADVISORIES: AgroAdvisory[] = [
  {
    advisoryId: 'adv-001',
    province: 'Kinshasa',
    commodity: null,
    severity: 'warning',
    title: '⚠️ Épisode de pluies intenses — Kinshasa',
    body: 'Des précipitations supérieures à 80mm/heure sont prévues du 15 au 18 août. Protégez vos sécheries et stockez vos récoltes à l\'abri. Evitez de planter pendant cette période.',
    source: 'METTELSAT · Direction Météorologique',
    effectiveFrom: new Date('2025-08-15'),
    effectiveTo: new Date('2025-08-18'),
  },
  {
    advisoryId: 'adv-002',
    province: 'all',
    commodity: 'Manioc',
    severity: 'info',
    title: '🐛 Alerte mosaïque du manioc',
    body: 'Des foyers de mosaïque du manioc ont été détectés dans plusieurs provinces. Utilisez exclusivement des boutures saines et certifiées. Arrachez et brûlez les plants infectés.',
    source: 'INERA · Institut National d\'Étude et de Recherche Agronomiques',
    effectiveFrom: new Date('2025-08-01'),
    effectiveTo: new Date('2025-09-30'),
  },
]
```

---

### UI Changes

#### `AgricultorHome.tsx` — `AgroAdvisoryCard`
Positioned **below quick actions, above AlertItem list**.

```
┌─────────────────────────────────────────────────────┐
│ ⚠️  Conseils agronomiques                   [Voir]  │
│ Épisode de pluies intenses — Kinshasa               │
│ Protégez vos sécheries... (truncated 2 lines)       │
└─────────────────────────────────────────────────────┘
```

- Severity border: critical = red (`var(--blocker)`), warning = amber (`var(--partial)`), info = blue (`var(--web)`)
- "Voir" → opens `AdvisoryDetailModal`
- If 2 advisories: show second as a smaller chip below
- If 0 advisories: card hidden (no empty state shown on home)

#### `AdvisoryDetailModal`
- Full title, body (multi-paragraph)
- Source + effective dates
- "Partager" button → `navigator.share()` or copy to clipboard
- "Fermer" button

---

## Admin Entry Point (mombongo-admin)
Simple "Publier un conseil" form (can be in `AdminDashboard` under a new "Agronomie" section):
- Title, Body, Province (select or "National"), Commodity (optional), Severity, effectiveFrom, effectiveTo
- Submit → `httpsCallable(functions, 'createAgroAdvisory')(payload)`

Add `createAgroAdvisory` CF (admin-only: verifies `users/{uid}.role === 'admin'`).

---

## Smoke Tests
1. AgricultorHome: `AgroAdvisoryCard` renders mock warning advisory with amber left border
2. Critical severity advisory: red left border (test by changing mock to `severity: 'critical'`)
3. Tap advisory → `AdvisoryDetailModal` opens with full text
4. No advisories (empty mock): card is hidden — no gap left in home layout
5. `sendCropStageAlerts` emulator test: culture `growthStage` transitions → FCM notification body matches `CROP_STAGE_TIPS`
6. `npx tsc --noEmit` passes in `mombongo-web`
