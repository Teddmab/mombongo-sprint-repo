# SP-05 — Critical Crash Fixes (3 Runtime Errors)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | SP-05 |
| Sprint | Sprint Patch 05 |
| Branch | `feature/sp-05-critical-crashes` |
| Merges into | `dev` |
| Estimate | 2h |
| Priority | BLOCKER — these cause runtime crashes visible to users |

---

## Context

Three screens have JavaScript runtime errors that crash the affected component:
- `BourseDetailScreen`: `<PaymentModal>` rendered but never imported → "PaymentModal is not defined"
- `AgricultorBourse`: `setVenteOpen` called inside module-scope `PriceRow` component but only
  declared inside `DesktopAgricultorBourse` → ReferenceError on any mobile bourse row tap
- `MerchantFinancement`: three wrong field names when reading farmer data from Firestore,
  causing `undefined` renders and TypeScript errors

---

## Bug 1 — `BourseDetailScreen.tsx` missing `<PaymentModal>` import

**Location**: `src/pages/BourseDetailScreen.tsx` line ~235  
**Error**: `ReferenceError: PaymentModal is not defined`  
**Trigger**: Any user taps "Investir" on a bourse detail screen on mobile

**Fix**: Add the missing import at the top of the file:
```tsx
import { PaymentModal } from "@/components/wallet/WalletModals";
```

Verify `PaymentModal` is exported from `WalletModals.tsx` (it should be — it's used elsewhere).

---

## Bug 2 — `AgricultorBourse.tsx` `setVenteOpen` ReferenceError

**Location**: `src/pages/bourse/AgricultorBourse.tsx` line ~36  
**Error**: `ReferenceError: setVenteOpen is not defined`  
**Trigger**: Any farmer taps a price row on mobile Bourse tab

**Root cause**: A module-level `PriceRow` component references `setVenteOpen`, but that state
is declared inside `DesktopAgricultorBourse` — a different function scope.

**Fix options** (pick the simplest):
- Pass `setVenteOpen` as a prop to `PriceRow`:
  ```tsx
  function PriceRow({ item, onSell }: { item: BourseTicker; onSell: () => void }) {
    // replace setVenteOpen() call with onSell()
  }
  // Usage: <PriceRow item={t} onSell={() => setVenteOpen(true)} />
  ```
- Or inline the content of `PriceRow` directly into the map where it's used.

---

## Bug 3 — `MerchantFinancement.tsx` wrong Firestore field names

**Location**: `src/pages/financement/MerchantFinancement.tsx` lines ~87, ~107, ~111

**Wrong → Correct** field names:
| Wrong | Correct | Source |
|-------|---------|--------|
| `farmer.photo` | `farmer.image` | `mock.ts` FarmerRequest type |
| `farmer.requestedAmount` | `farmer.needed` | same |
| `farmer.funded` | `farmer.raised` | same |

**Fix**: Update the three field access expressions:
```tsx
// line ~87
<img src={farmer.image} ...>

// line ~107
<span>{farmer.needed} USD demandés</span>

// line ~111
<div style={{ width: `${(farmer.raised / farmer.needed) * 100}%` }} />
```

When wiring to real Firestore data (SG-03), the CF response fields will need to match. For now,
align with what the mock data provides so the component renders correctly in dev mode.

---

## Acceptance Criteria
- [ ] `/bourse/:id` on mobile no longer throws "PaymentModal is not defined"
- [ ] Farmer tapping a price row on mobile bourse no longer throws "setVenteOpen is not defined"
- [ ] `MerchantFinancement` renders farmer image, requested amount, and progress bar correctly in dev mode
- [ ] `npx tsc --noEmit` passes
- [ ] `npx vitest run` passes

---

## Implementation Status (updated 2026-07-23)

**NOT DONE — 3 confirmed runtime crashes**

### Bug 1 — BourseDetailScreen: PaymentModal not imported ❌ CONFIRMED
- `<PaymentModal>` is rendered in the JSX but there is no import for it at the top of the file
- `BourseInvestModal` is imported instead — `PaymentModal` is from `@/components/wallet/WalletModals`
- **Fix needed**: Add `import { PaymentModal } from "@/components/wallet/WalletModals"` to `BourseDetailScreen.tsx`

### Bug 2 — AgricultorBourse: setVenteOpen closure bug ❌ CONFIRMED  
- Line 39: `PriceRow` (module-scope component) calls `setVenteOpen(true)` 
- `setVenteOpen` is declared inside `DesktopAgricultorBourse` (line 52) — different scope
- **Fix needed**: Pass `onSell: () => void` prop to `PriceRow`, replace `setVenteOpen()` with `onSell()`

### Bug 3 — MerchantFinancement: wrong field names ❌ CONFIRMED
- Line 86–87: `farmer.photo` → should be `farmer.image`
- Line 106: `farmer.requestedAmount` → should be `farmer.needed`
- Lines 110–111, 162–163, 167: `farmer.funded` → should be `farmer.raised`
