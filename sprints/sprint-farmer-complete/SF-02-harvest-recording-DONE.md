# SF-02 — Harvest Recording & Actual Yield Tracking

## Goal
After harvest month is reached, the farmer logs the actual outcome: `actualYieldKg`, quality grade (A/B/C/D), harvest date, and optional warehouse location. The recorded harvest auto-links to SF-01 (Transformer) and S8-01 (market listing) CTAs, closing the harvest loop and feeding SF-03 analytics.

## Status
**TODO** · Priority P1 · Est. 4h · Web

## Sprint file
`sprint-farmer-complete/SF-02-harvest-recording.md`

## Branch
`feature/sf-02-harvest-recording`

## Repos
- `mombongo-web`
- `mombongo-functions`

---

## Firestore

### Collection: `harvest_records`
```
harvest_records/{recordId}
  farmerId: string               // uid of caller
  cultureId: string              // ref to cultures/{id}
  exploitationId: string         // ref to exploitations/{id}
  commodity: string              // e.g. "Manioc"
  expectedYieldKg: number        // from cultures/{id}.expectedYieldKg
  actualYieldKg: number
  yieldVsExpectedPct: number     // computed: (actualYieldKg / expectedYieldKg) * 100
  qualityGrade: 'A' | 'B' | 'C' | 'D'
  harvestDate: Timestamp
  warehouseLocation?: string
  notes?: string
  createdAt: Timestamp
```

---

## Cloud Functions

### `recordActualHarvest`
**Callable** · farmerId inferred from auth

**Input:**
```typescript
{
  cultureId: string
  actualYieldKg: number          // must be > 0
  qualityGrade: 'A' | 'B' | 'C' | 'D'
  harvestDate: string            // ISO date
  warehouseLocation?: string
  notes?: string
}
```

**Logic:**
1. Auth check — `uid` required
2. Fetch `cultures/{cultureId}` — must belong to caller
3. Compute `yieldVsExpectedPct = (actualYieldKg / culture.expectedYieldKg) * 100`
4. Create `harvest_records/{recordId}`
5. Update `cultures/{cultureId}`: `actualYieldKg`, `yieldVsExpectedPct`, `status → "harvested"`, `harvestDate`
6. Return `{ recordId, yieldVsExpectedPct }`

**Errors:** `unauthenticated`, `not-found` (culture not found or wrong owner), `invalid-argument` (actualYieldKg ≤ 0)

---

### `getHarvestHistory`
**Callable** · farmerId inferred from auth

**Input:**
```typescript
{
  exploitationId?: string        // filter by exploitation; omit for all
}
```

**Output:** Array of `harvest_records` ordered by `harvestDate` desc

---

## Frontend

### Hook: `src/hooks/useHarvestRecords.ts`
```typescript
export function useMyHarvestRecords(exploitationId?: string): HarvestRecord[]
export function useRecordActualHarvest(): UseMutationResult<...>
```

Mock guard:
```typescript
if (isDevMode()) return [MOCK_HARVEST_RECORDS]
```

`MOCK_HARVEST_RECORDS`:
```typescript
[{
  recordId: 'hr-001',
  cultureId: 'c-001',
  commodity: 'Manioc',
  expectedYieldKg: 800,
  actualYieldKg: 720,
  yieldVsExpectedPct: 90,
  qualityGrade: 'B',
  harvestDate: new Date('2025-06-15'),
}]
```

---

### UI Changes

#### `CultureCard.tsx`
- Condition: `moisRecolte === currentMonth && culture.status !== 'harvested'`
- Add button: **"🌾 Enregistrer la récolte"** → opens `EnregistrerRecolteModal`

#### `EnregistrerRecolteModal`
2-step modal:

**Step 1:**
- `actualYieldKg` number input (label: "Rendement réel (kg)")
- `qualityGrade` segmented control: A / B / C / D (A = excellent, D = mediocre)
- `harvestDate` date picker (default: today)

**Step 2:**
- `warehouseLocation` text input (optional)
- `notes` textarea (optional)
- Auto-display: **"Rendement vs prévision: +10%"** in green/red pill

**Success screen** (after submit):
- "✅ Récolte enregistrée"
- **"🔄 Transformer ce produit"** → opens SF-01 NouvelleTransformationModal (pre-fills cultureId + commodity)
- **"📢 Publier sur le marché"** → opens S8-01 PublierProduitModal (pre-fills commodity + quantityKg)

#### Mon Exploitation — Harvest History Section
- Shows `harvest_records` for the exploitation
- Columns: Commodity, Date, Quantity, Quality grade badge (A=green, B=yellow, C=orange, D=red), Yield vs expected pill

---

## Smoke Tests
1. CultureCard: `moisRecolte === currentMonth` → **"🌾 Enregistrer la récolte"** visible, not visible for other months
2. 2-step modal: submit → `harvest_records/{id}` created in Firestore + `cultures/{id}.status === "harvested"`
3. Post-record success screen: SF-01 and S8-01 CTAs both visible
4. Mon Exploitation: harvest history row appears after recording
5. `npx tsc --noEmit` passes in `mombongo-web`
