# SFA-01-05 — Transactions History Screen (Farmer App)

## Context
Port of SM-01-01. Farmers need to see their wallet transaction history: deposits (from financing disbursements, harvest sales), withdrawals (to bank/mobile money), and fees. The web has a transaction history in the wallet area. Mobile has no dedicated transactions screen — the wallet modal shows a balance but no history.

CF available: `getTransactions` — deployed as part of SG-06 (investor portfolio) and uses the same transactions collection. For farmers the query filters by `userId` and shows all transaction types.

## Scope
- Create `src/hooks/useTransactions.ts` — calls `getTransactions` CF
- Create `src/screens/wallet/TransactionsScreen.tsx`
- Add a "Voir l'historique" link from the wallet balance row in ProfileScreen
- Transaction types: `deposit`, `withdrawal`, `financing_disbursement`, `harvest_sale`, `fee`
- Show running balance after each transaction
- Filter chips: Tout | Entrées | Sorties

## Cloud Function required
`getTransactions` — already deployed (SG-06). Input: `{ limit?: number; type?: string }` → output: `{ transactions: Transaction[] }`

## Files to create
- `src/hooks/useTransactions.ts`
- `src/screens/wallet/TransactionsScreen.tsx`

## Implementation

### `src/hooks/useTransactions.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

export type Transaction = {
  id: string
  type: 'deposit' | 'withdrawal' | 'financing_disbursement' | 'harvest_sale' | 'fee'
  amountCdf: number
  direction: 'in' | 'out'
  description: string
  createdAt: string
  balanceAfterCdf?: number
}

const MOCK_TRANSACTIONS: Transaction[] = [
  { id: '1', type: 'financing_disbursement', amountCdf: 250_000, direction: 'in', description: 'Décaissement prêt agricole', createdAt: new Date().toISOString(), balanceAfterCdf: 375_000 },
  { id: '2', type: 'withdrawal', amountCdf: 50_000, direction: 'out', description: 'Retrait M-Pesa', createdAt: new Date(Date.now() - 86400000).toISOString(), balanceAfterCdf: 125_000 },
  { id: '3', type: 'harvest_sale', amountCdf: 180_000, direction: 'in', description: 'Vente 400kg Maïs', createdAt: new Date(Date.now() - 172800000).toISOString(), balanceAfterCdf: 175_000 },
]

export function useTransactions(type?: string) {
  return useQuery({
    queryKey: ['transactions', type],
    queryFn: async () => {
      if (isDevMode()) return MOCK_TRANSACTIONS
      const res = await httpsCallable<{ limit?: number; type?: string }, { transactions: Transaction[] }>(
        functions, 'getTransactions'
      )({ limit: 50, type })
      return res.data.transactions
    },
    staleTime: 5 * 60_000,
  })
}
```

### TransactionsScreen.tsx
```typescript
// Filter chips at top: Tout | Entrées | Sorties
// Transaction list, newest first:
//   Each row: type icon + description + amount (green for in, red for out) + date
//   Running balance shown as subtext if available
//   Grouped by date (Today / Yesterday / This week / Older)
// Empty state: "Aucune transaction pour le moment"
// Pull to refresh

const TYPE_ICONS: Record<Transaction['type'], string> = {
  deposit: '💰',
  withdrawal: '📤',
  financing_disbursement: '🏦',
  harvest_sale: '🌾',
  fee: '📋',
}
```

## Acceptance criteria
- [ ] Transactions screen accessible from Profile wallet balance row
- [ ] List loads from `getTransactions` CF
- [ ] "Entrées" filter shows only `direction: 'in'` transactions
- [ ] "Sorties" filter shows only `direction: 'out'`
- [ ] Running balance shown per transaction when available
- [ ] Pull to refresh works
- [ ] Dev mode shows 3 mock transactions

## Smoke test
1. Open Profile → tap "Voir l'historique" → transactions screen loads
2. Switch filter to "Entrées" → only green amounts visible
3. Switch to "Sorties" → only red amounts visible
4. Pull to refresh → no crash
5. In live mode: real transactions from CF load correctly
