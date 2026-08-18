# SFA-05-02 — Score Unlock Thresholds (Farmer App)

## Context
Port of web sprint SU-03-04. The Mombongo Score (0–100) gates access to premium features. This sprint adds `ScoreGate` component to the mobile app that shows a "score too low" overlay when a farmer tries to access a locked feature.

Unlock thresholds (same as web):
- Score ≥ 20: access Bourse listing (publish a listing)
- Score ≥ 40: access AgroExchange (place orders)
- Score ≥ 60: access premium financing (larger loan amounts)
- Score ≥ 80: access cooperative sales (SF-06 feature)

## Scope
- Create `src/components/ScoreGate.tsx` — wrapper component; shows lock overlay when score below threshold
- Wrap relevant screens/buttons with `ScoreGate`
- Show "Votre score: X/100 · Il vous faut X points de plus" with a mini progress bar
- Link to Score detail breakdown

## Files to create / modify
- `src/components/ScoreGate.tsx`
- `src/screens/bourse/FarmerBourseScreen.tsx` — wrap "Publier" FAB with ScoreGate(20)
- `src/screens/exchange/AgroExchangeScreen.tsx` — wrap "Je veux vendre" with ScoreGate(40)

## Implementation

### ScoreGate.tsx
```typescript
type Props = {
  minScore: number
  children: React.ReactNode
  featureName: string
}

export function ScoreGate({ minScore, children, featureName }: Props) {
  const { data: scoreData } = useMomBongoScore()
  const score = scoreData?.score ?? 0

  if (score >= minScore) return <>{children}</>

  return (
    <TouchableOpacity onPress={() => Alert.alert(
      `${featureName} verrouillé`,
      `Votre score: ${score}/100\nIl vous faut ${minScore - score} points de plus.\n\nAméliorez votre score en complétant votre profil, formant sur Academia et publiant vos récoltes.`
    )}>
      <View style={styles.lockedOverlay}>
        {children}
        <View style={styles.lockBadge}>
          <Text>🔒 Score {minScore}+</Text>
        </View>
      </View>
    </TouchableOpacity>
  )
}
```

### Bourse FAB wrapper
```typescript
<ScoreGate minScore={20} featureName="Publication Bourse">
  <FAB icon="plus" onPress={() => setShowCreateSheet(true)} />
</ScoreGate>
```

## Development override
Add `?mockScore=85` URL param equivalent: in dev mode, set `MOCK_SCORE_OVERRIDE` env var or a debug toggle in developer settings to bypass all gates (for testing locked features without grinding score).

## Acceptance criteria
- [ ] Farmer with score < 20 sees lock overlay on Bourse publish FAB
- [ ] Tapping locked feature shows score breakdown alert
- [ ] Farmer with score ≥ threshold can access the feature normally
- [ ] `ScoreGate` handles loading state (no flash of locked state when score is already high)

## Smoke test
1. In dev mode with MOCK_SCORE = 10: open Bourse → FAB shows lock badge
2. Tap locked FAB → alert shows "Score 20+ requis"
3. Change MOCK_SCORE to 50: FAB no longer locked; tapping opens CreateListingSheet
4. Open AgroExchange with MOCK_SCORE = 30 → sell button locked
5. Change to 50 → sell button unlocked
