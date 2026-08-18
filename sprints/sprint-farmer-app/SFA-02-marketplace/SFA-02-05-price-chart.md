# SFA-02-05 — Price Trend Chart (Farmer App)

## Context
Port of SM-06-02. In the AgroExchange screen (SFA-02-03) and the Market Prices screen (SFA-01-03), showing a 30-day price trend chart for a commodity gives farmers timing intuition ("prices are rising — should I wait to sell?"). The web market intelligence dashboard (S9-02) already has a price chart powered by `getMarketAnalytics` CF.

## Scope
- Create `src/components/PriceTrendChart.tsx` — 30-day line chart using `react-native-svg` (already installed for StreakWidget)
- Add the chart to `FarmerMarketScreen.tsx` when a commodity is selected (tap on a price card → expand chart)
- Add the chart to `AgroExchangeScreen.tsx` in the commodity tab header

## Cloud Function required
`getMarketAnalytics` — deployed in S9-02. Input: `{ commodity: string; province?: string; days?: number }` → output: `{ dataPoints: Array<{ date: string; pricePerKgCdf: number }> }`

## Files to create
- `src/components/PriceTrendChart.tsx`
- `src/hooks/usePriceTrend.ts`

## Implementation

### `src/hooks/usePriceTrend.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

type DataPoint = { date: string; pricePerKgCdf: number }

const MOCK_DATA: DataPoint[] = Array.from({ length: 30 }, (_, i) => ({
  date: new Date(Date.now() - (29 - i) * 86400000).toISOString(),
  pricePerKgCdf: 400 + Math.round(Math.sin(i / 5) * 40 + Math.random() * 20),
}))

export function usePriceTrend(commodity: string, province?: string) {
  return useQuery({
    queryKey: ['priceTrend', commodity, province],
    queryFn: async () => {
      if (isDevMode()) return MOCK_DATA
      const res = await httpsCallable<{ commodity: string; province?: string; days: number }, { dataPoints: DataPoint[] }>(
        functions, 'getMarketAnalytics'
      )({ commodity, province, days: 30 })
      return res.data.dataPoints
    },
    staleTime: 60 * 60_000,
  })
}
```

### PriceTrendChart.tsx (SVG line chart)
```typescript
import Svg, { Path, Line, Text as SvgText, Circle } from 'react-native-svg'

export function PriceTrendChart({ data, commodity }: { data: DataPoint[]; commodity: string }) {
  const WIDTH = 320
  const HEIGHT = 120
  const PADDING = { left: 40, right: 10, top: 10, bottom: 24 }

  const prices = data.map(d => d.pricePerKgCdf)
  const minP = Math.min(...prices)
  const maxP = Math.max(...prices)

  // Map data points to SVG coordinates
  const toX = (i: number) => PADDING.left + (i / (data.length - 1)) * (WIDTH - PADDING.left - PADDING.right)
  const toY = (p: number) => PADDING.top + (1 - (p - minP) / (maxP - minP)) * (HEIGHT - PADDING.top - PADDING.bottom)

  // Build SVG path
  const pathD = data.map((d, i) => `${i === 0 ? 'M' : 'L'}${toX(i)},${toY(d.pricePerKgCdf)}`).join(' ')

  // Last point highlight
  const lastX = toX(data.length - 1)
  const lastY = toY(prices[prices.length - 1])

  return (
    <View>
      <Text style={styles.chartTitle}>{commodity} · 30 derniers jours</Text>
      <Svg width={WIDTH} height={HEIGHT}>
        {/* Y-axis labels */}
        <SvgText x={PADDING.left - 4} y={PADDING.top + 4} textAnchor="end" fontSize={9}>{maxP}FC</SvgText>
        <SvgText x={PADDING.left - 4} y={HEIGHT - PADDING.bottom} textAnchor="end" fontSize={9}>{minP}FC</SvgText>
        {/* Line */}
        <Path d={pathD} stroke="#22C55E" strokeWidth={2} fill="none" />
        {/* Current price dot */}
        <Circle cx={lastX} cy={lastY} r={4} fill="#22C55E" />
        <SvgText x={lastX + 6} y={lastY + 4} fontSize={10} fill="#22C55E">{prices[prices.length - 1]}FC</SvgText>
      </Svg>
    </View>
  )
}
```

### Integration in FarmerMarketScreen
```typescript
const [selectedCommodity, setSelectedCommodity] = useState<string | null>(null)
const { data: trendData } = usePriceTrend(selectedCommodity ?? '', undefined)

// When farmer taps a price card: expand chart below it
{selectedCommodity && trendData && (
  <PriceTrendChart data={trendData} commodity={selectedCommodity} />
)}
```

## Acceptance criteria
- [ ] Tapping a commodity price card in FarmerMarketScreen expands a 30-day chart below it
- [ ] Chart shows correct shape (prices going up/down visually)
- [ ] Current price highlighted as a dot with label
- [ ] Min/max Y-axis labels shown
- [ ] Dev mode shows sinusoidal mock data (visible curve)
- [ ] AgroExchange commodity tab header also shows chart for selected commodity

## Smoke test
1. Open Market tab → tap "Maïs" price card → chart expands below
2. Confirm chart renders with a visible line (not flat/empty)
3. Tap another commodity → chart updates
4. Open AgroExchange → select "Maïs" tab → confirm chart in header
5. In live mode: real 30-day price history loads from CF
