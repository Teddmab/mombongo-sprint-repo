# SF-06 — Vente Groupée — Cooperative / Group Sales

## Goal
Multiple farmers pool produce to hit minimum lot sizes required by large buyers (flour mills, export houses, food processors). Each farmer contributes a portion. The lot creator negotiates the contract. Revenue split pro-rata by contribution on delivery. Major unlock for smallholder farmers who individually can't meet buyer minimums — critical for DRC market access.

## Status
**TODO** · Priority P2 · Est. 6h · Web

## Branch
`feature/sf-06-cooperative-sales`

## Repos
- `mombongo-web`
- `mombongo-functions`

---

## Firestore

### Collection: `cooperative_lots`
```
cooperative_lots/{lotId}
  creatorFarmerId: string
  commodity: string
  totalTargetKg: number          // minimum needed to attract buyers
  currentKg: number              // live sum of members[].contributionKg
  pricePerKgCdf: number          // proposed asking price
  province: string
  territory: string
  status: 'open' | 'full' | 'contracted' | 'completed'
  members: [{
    farmerId: string
    displayName: string          // denormalized for display
    contributionKg: number
    confirmed: boolean           // true once farmer has confirmed delivery
    joinedAt: Timestamp
  }]
  contractId?: string            // ref to bourse_contracts when lot becomes a bourse opportunity
  deadline: Timestamp
  description?: string
  createdAt: Timestamp
```

---

## Cloud Functions

### `createCooperativeLot` (callable)
**Input:**
```typescript
{
  commodity: string
  totalTargetKg: number          // must be ≥ 100
  pricePerKgCdf: number
  province: string
  territory: string
  deadline: string               // ISO date, must be in the future
  myContributionKg: number       // creator's own contribution (must be > 0)
  description?: string
}
```
**Logic:**
1. Auth + uid check
2. Validate: `totalTargetKg >= 100`, `deadline > now`, `myContributionKg > 0 && myContributionKg <= totalTargetKg`
3. Fetch caller displayName from `users/{uid}`
4. Create `cooperative_lots/{lotId}`: `currentKg = myContributionKg`, `members = [{ farmerId: uid, displayName, contributionKg: myContributionKg, confirmed: false, joinedAt: now }]`, `status = 'open'`
5. Return `{ lotId }`

---

### `joinCooperativeLot` (callable)
**Input:**
```typescript
{
  lotId: string
  contributionKg: number         // must be > 0
}
```
**Logic:**
1. Auth check
2. Fetch lot — must exist and `status === 'open'`, caller must not already be in `members[]`
3. Validate: `contributionKg > 0`, `currentKg + contributionKg <= totalTargetKg * 1.1` (allow 10% overflow)
4. `arrayUnion` new member, increment `currentKg`
5. If `currentKg >= totalTargetKg` → `status = 'full'`; send FCM to all members: "Lot complet — votre vente groupée de Manioc est prête!"
6. Return `{ currentKg, status }`

---

### `getCooperativeLots` (callable)
**Input:**
```typescript
{
  commodity?: string
  province?: string
}
```
**Output:** Active (`status === 'open'`) `cooperative_lots`, ordered by `createdAt` desc, limit 50.

---

### `getMyCooperativeLots` (callable)
**Output:** Lots where `creatorFarmerId === uid` OR `uid` appears in `members[].farmerId`, ordered by `createdAt` desc.

---

## Frontend

### Hook: `src/hooks/useCooperativeLots.ts`
```typescript
export function useOpenCooperativeLots(commodity?: string, province?: string): CooperativeLot[]
export function useMyCooperativeLots(): CooperativeLot[]
export function useCreateCooperativeLot(): UseMutationResult<...>
export function useJoinCooperativeLot(): UseMutationResult<...>
```

Mock data:
```typescript
const MOCK_LOTS: CooperativeLot[] = [
  {
    lotId: 'cl-001', commodity: 'Manioc', totalTargetKg: 2000, currentKg: 1200,
    pricePerKgCdf: 350, province: 'Kinshasa', territory: 'Lukunga',
    status: 'open', members: [
      { farmerId: 'u-001', displayName: 'Jean Mutombo', contributionKg: 700 },
      { farmerId: 'u-002', displayName: 'Marie Kalala',  contributionKg: 500 },
    ],
    deadline: new Date('2025-08-30'),
  },
]
```

---

### UI

**Location:** New **"Vente groupée"** tab in `AgricultorBourse.tsx` (alongside "Mes cultures" / "Tous les prix")

#### Open Lots List
`CoopLotCard` layout:
```
[ 🌾 Manioc ]  Province: Kinshasa · Lukunga
Objectif: 2,000 kg   ████████░░  60% (1,200 / 2,000 kg)
Prix demandé: 350 CDF/kg    Clôture: 30 août 2025
2 agriculteurs  |  [ Rejoindre ]
```

- "Rejoindre" → `RejoindreCoopLotModal`: contributionKg input, "Ma contribution: X kg" summary
- If caller already in `members[]` → "Vous participez (700 kg)" chip, no Join button

#### My Lots List
- Same `CoopLotCard` but shows **"Modifier"** and **"Annuler"** for lots I created (if `status === 'open'`)
- Lot detail: members list with contributions, confirmation status

#### `CreateCoopLotModal`
- Commodity (select from useMyCultures + free text)
- Objectif total (kg), Prix demandé (CDF/kg), Province, Territoire, Date limite
- Ma contribution (kg)
- Description (optional)
- Live preview: "Vous contribuez 700 kg sur 2,000 kg (35%)"

---

## Smoke Tests
1. Bourse: "Vente groupée" tab renders — open lots list shows 1 mock lot
2. `CoopLotCard`: progress bar at 60%, member count shows 2
3. Create lot modal → submit → `cooperative_lots/{lotId}` created in Firestore
4. Second farmer joins → `members[]` has 2 entries, `currentKg` updated
5. Lot reaches `totalTargetKg` → `status = 'full'`, progress bar 100%, Join button replaced by chip
6. `npx tsc --noEmit` passes in `mombongo-web`
