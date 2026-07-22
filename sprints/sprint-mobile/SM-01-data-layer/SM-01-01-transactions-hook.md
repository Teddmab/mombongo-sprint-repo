# SM-01-01 — useTransactions hook + transaction history screen

**Sprint:** SM-01 · Data layer  
**Branch:** `feature/sm-01-data-layer`

## Context
The web app shows a full transaction history in the Portfolio/Profile area. Mobile has `hooks/useTransactions.ts` but it may not be wired to a visible screen. Users cannot see their deposit/withdrawal/investment history on mobile.

## Acceptance criteria
- [ ] `hooks/useTransactions.ts` calls `httpsCallable(functions, "getTransactions")` when not in devMode
- [ ] Mock fallback returns `MOCK_TRANSACTIONS` from `data/mock.ts` (add if missing)
- [ ] A `TransactionsScreen` (or section inside `PortfolioScreen`) renders a flat list of transactions
- [ ] Each row shows: type icon, amount, status pill, formatted date
- [ ] Reachable from `app/portfolio.tsx` via a "Voir toutes les transactions" link
- [ ] `app/transactions.tsx` route exists (or nested under portfolio)

## Data shape
```ts
interface Transaction {
  id: string;
  type: "investment" | "deposit" | "withdrawal" | "financing" | "bourse_investment";
  amountUsd: number;
  currency: "USD" | "CDF";
  status: "completed" | "pending" | "failed";
  createdAt: { seconds: number };
  description?: string;
}
```

## Implementation notes
- CF: `getTransactions` — same function used by web; returns `{ transactions: Transaction[] }`
- Status pill: completed = green, pending = amber, failed = red
- Pull to refresh with `useQuery` `refetch`
