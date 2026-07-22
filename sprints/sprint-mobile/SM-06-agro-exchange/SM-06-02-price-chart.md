# SM-06-02 — AgroExchange: commodity price chart

**Sprint:** SM-06 · AgroExchange  
**Branch:** `feature/sm-06-agro-exchange`

## Context
Users need price history to make informed trading decisions. This story adds a price chart to the commodity detail view.

## Acceptance criteria
- [ ] Tapping a commodity row opens `AgroExchangeDetailScreen` (new route `app/agro-exchange/[listingId].tsx`)
- [ ] Detail screen shows: commodity name, current price, 24h change, 7d chart
- [ ] Chart: line chart using `react-native-svg` + a simple SVG path or `victory-native`
- [ ] Time range selector: 7D / 30D / 90D
- [ ] `useAgroPriceHistory(commodityId, range)` hook: `httpsCallable(functions, "getAgroPriceHistory")({ commodityId, range })`
- [ ] Price history returns: `{ points: { date: string; priceUsd: number }[] }`
- [ ] In devMode, returns mock price points with realistic variation
- [ ] Buy/Sell CTA at the bottom of the detail screen (links to AgroOrderModal)

## Implementation notes
- `react-native-svg` is likely not yet installed — add to package.json
- For the chart, a simple SVG polyline is sufficient for MVP
- "Coming from BourseTickerBar" — BourseTickerBar shows commodity prices; deep-link to AgroExchange detail
