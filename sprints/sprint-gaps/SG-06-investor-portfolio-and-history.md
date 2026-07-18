# SG-06 — Investor: Real Portfolio Chart + Transaction History

## Why this matters
Investors are the revenue-generating users. They can invest (wired) but they cannot see real performance data, their full history, or portfolio breakdown. The home screen chart is hardcoded. The activity feed is mock.

## Current state
- `HomeScreen.tsx`: `PORTFOLIO_TREND` hardcoded array (lines 46–49); time-period buttons (7j / 1m / 3m / 1a) are display-only with no effect
- Activity feed on home uses mock `activity` data from `src/data/mock.ts`
- Bourse "Mes positions" button has no onClick handler
- No dedicated transaction history screen
- No investment breakdown by crop or region

## Work items

### 1. Real portfolio chart
- CF: `getPortfolioTrend(uid, period)` — computes running wallet balance over the requested period from `transactions` history
  - `period`: '7d' | '30d' | '90d' | '1y'
  - Returns: `{ date: string, balanceUsd: number }[]`
- Wire `usePortfolioTrend(period)` hook → `httpsCallable(functions, 'getPortfolioTrend')({ period })`
- Time-period buttons now update `period` state → refetch chart data
- Use the existing `LineChart` (Recharts) already in the component

### 2. Real activity feed on Home
- CF: `getMyActivity(uid, limit)` — returns last N transactions/investments combined, orderBy createdAt desc
- Replace mock `activity` array in `HomeScreen` with real query
- Show: type icon, description, amount, date — nothing more

### 3. Transaction history screen (`/historique`)
- New screen accessible from Home "Voir tout" link next to the activity feed
- Full paginated list of all transactions for the logged-in user
- Filter by type: dépôt / retrait / investissement / revenu / tous
- Each row: icon, description, amount (green for credit, red for debit), date
- Tap row → simple detail view (status, reference ID, method)
- CF: `getMyTransactions(uid, { type?, limit?, cursor? })`

### 4. Bourse "Mes positions" screen
- "Mes positions" button on BourseScreen navigates to `/bourse/mes-positions`
- Lists all `bourse_investments` where `investorId == uid`
- Each row: opportunity title, amount invested, current status, expected return
- CF: `getMyBoursePositions(uid)`

### 5. Investment breakdown widget (Home)
- Small section below the chart: "Répartition de mon portefeuille"
- Simple horizontal bar or pie: by crop type (maïs X%, manioc Y%, riz Z%)
- CF: `getPortfolioBreakdown(uid)` — aggregates `investments` by `cropType`

## Cloud Functions needed
- `getPortfolioTrend(uid, period)` — balance history over time
- `getMyActivity(uid, limit)` — recent mixed transactions + investments
- `getMyTransactions(uid, filters)` — full transaction history with pagination
- `getMyBoursePositions(uid)` — active bourse investment positions
- `getPortfolioBreakdown(uid)` — portfolio split by crop type

## Acceptance criteria
- [ ] Portfolio chart shows real balance history from Firestore
- [ ] Time-period buttons change chart data
- [ ] Activity feed on Home shows real last 5 transactions/investments
- [ ] Transaction history screen is accessible and paginated
- [ ] Bourse "Mes positions" button navigates to a real screen with real data
- [ ] Portfolio breakdown widget shows by crop type
