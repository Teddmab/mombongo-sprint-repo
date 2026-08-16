# SF-05 — Farm Inputs & Expense Tracker

## Goal
Record farm inputs per culture per growth stage: seeds (variety + kg), fertilizer (type + kg), pesticide (product + L), labor (days × daily rate CDF), equipment rental. Costs flow automatically into SF-03 P&L. Closes the loop between cultural calendar stages and real cash spent per crop.

## Status
**TODO** · Priority P2 · Est. 4h · Web

## Branch
`feature/sf-05-farm-inputs`

## Repos
- `mombongo-web`
- `mombongo-functions`

---

## Firestore

### Collection: `farm_inputs`
```
farm_inputs/{inputId}
  farmerId: string
  cultureId: string
  exploitationId: string
  inputType: 'semences' | 'engrais' | 'pesticide' | 'main-oeuvre' | 'location-equipement' | 'autre'
  inputName: string              // e.g. "Urée 50kg", "Glyphosate 1L", "Labour manuel"
  quantityKg?: number            // for seeds/fertilizer
  unitCount?: number             // for labor (days) or equipment (rental days)
  costCdf: number                // total cost in CDF (must be > 0)
  recordedAt: Timestamp
  growthStage?: string           // optional link to cultural calendar stage
  notes?: string
  createdAt: Timestamp
```

---

## Cloud Functions

### `recordFarmInput` (callable)
**Input:**
```typescript
{
  cultureId: string
  inputType: InputType
  inputName: string
  costCdf: number                // must be > 0
  quantityKg?: number
  unitCount?: number
  growthStage?: string
  notes?: string
}
```
**Logic:**
1. Auth check — uid required
2. Fetch `cultures/{cultureId}` — must belong to caller
3. Create `farm_inputs/{inputId}` with `farmerId = uid`, `exploitationId` from culture
4. Return `{ inputId }`

**Errors:** `unauthenticated`, `not-found`, `invalid-argument` (costCdf ≤ 0 or missing inputName)

---

### `getFarmInputs` (callable)
**Input:**
```typescript
{
  cultureId?: string             // filter by culture; omit for all cultures of caller
  season?: string                // optional date range filter
}
```
**Output:** Array of `farm_inputs` ordered by `recordedAt` desc, with `totalCostCdf` sum appended.

---

### `deleteFarmInput` (callable)
**Input:** `{ inputId: string }`
**Logic:** Ownership check (`farmerId === uid`), hard delete.

---

## Frontend

### Hook: `src/hooks/useFarmInputs.ts`
```typescript
export function useFarmInputs(cultureId?: string): { inputs: FarmInput[]; totalCostCdf: number }
export function useRecordFarmInput(): UseMutationResult<...>
export function useDeleteFarmInput(): UseMutationResult<...>
```

Mock:
```typescript
const MOCK_INPUTS: FarmInput[] = [
  { inputId: 'fi-001', cultureId: 'c-001', inputType: 'semences',   inputName: 'Semences Manioc local', quantityKg: 50,  costCdf: 25000, growthStage: 'Plantation' },
  { inputId: 'fi-002', cultureId: 'c-001', inputType: 'engrais',    inputName: 'NPK 17-17-17',          quantityKg: 25,  costCdf: 40000, growthStage: 'Croissance' },
  { inputId: 'fi-003', cultureId: 'c-001', inputType: 'main-oeuvre',inputName: 'Désherbage manuel',     unitCount: 5,    costCdf: 30000, growthStage: 'Entretien' },
]
// totalCostCdf = 95,000
```

---

### UI

#### Culture Detail Screen (inside Mon Exploitation)
Add **"Intrants & Dépenses"** tab alongside existing tabs:

**Tab content:**
- Inputs grouped by `inputType` with section headers (icons: 🌱 Semences, 🧪 Engrais, 🚿 Pesticides, 👷 Main-d'œuvre, 🚜 Équipement, 📦 Autre)
- Each group shows: input name, quantity/unit, cost in CDF, growth stage badge, delete button
- **Subtotals per group** + **Grand total** sticky footer
- Empty state: "Aucun intrant enregistré" + "Ajouter" CTA
- FAB: **"＋ Ajouter un intrant"** → `NouvelIntrantModal`

#### `NouvelIntrantModal`
- **Type** — grid of 6 icon buttons (Semences, Engrais, Pesticide, Main-d'œuvre, Équipement, Autre)
- **Nom** — text input (placeholder varies by type: "Urée 50kg", "Désherbage 5 jours", etc.)
- **Quantité** — conditional: kg for semences/engrais, jours for main-d'œuvre (optional)
- **Coût total (CDF)** — number input
- **Stade** — optional dropdown from cultural calendar stages
- **Notes** — optional

#### `CultureCard.tsx`
- Show total input cost chip: **"Intrants: 95,000 CDF"** below crop info (from `useFarmInputs(cultureId).totalCostCdf`)

---

## Smoke Tests
1. Culture detail: "Intrants & Dépenses" tab loads with 3 mock inputs grouped by type
2. `NouvelIntrantModal`: fill type + name + costCdf → submit → `farm_inputs/{id}` created, total updates
3. Delete a row → confirmation → row disappears
4. CultureCard: shows "Intrants: 95,000 CDF" chip in dev mode
5. `npx tsc --noEmit` passes in `mombongo-web`
