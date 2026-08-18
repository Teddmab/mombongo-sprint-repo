# SFA-07-02 — Premium Content Gating (Farmer App)

## Context
Port of SM-07-01. Certain features and content are locked behind paid subscription tiers. This sprint adds a `PlanGate` component (analogous to `ScoreGate` from SFA-05-02) and applies it to gated features.

**Gated features by plan:**

| Feature | Min plan |
|---------|----------|
| All Academia courses (beyond first 2) | Standard |
| Price trend chart (SFA-02-05) | Standard |
| "Pour vous" buyer matching (SFA-02-02) | Standard |
| Financing applications > 250 000 FC | Standard |
| Priority Bourse listing | Premium |
| Dedicated agent assignment | Premium |
| Financing applications > 1 000 000 FC | Premium |

## Scope
- Create `src/components/PlanGate.tsx` — wrapper showing upgrade prompt when plan insufficient
- Apply `PlanGate` to all gated features listed above
- `PlanGate` reads current plan from `useSubscription()` (already fetched and cached)
- Upgrade prompt: bottom sheet with plan comparison + "Passer à Standard/Premium" button → SubscriptionScreen

## Files to create / modify
- `src/components/PlanGate.tsx`
- Apply in: AcademiaScreen, PriceTrendChart, BuyerMatchCard, ApplyForFinancingSheet, CreateListingSheet

## Implementation

### PlanGate.tsx
```typescript
const PLAN_ORDER: Record<string, number> = { free: 0, standard: 1, premium: 2 }

type Props = {
  minPlan: 'standard' | 'premium'
  featureName: string
  children: React.ReactNode
}

export function PlanGate({ minPlan, featureName, children }: Props) {
  const { data: subscription } = useSubscription()
  const currentPlanOrder = PLAN_ORDER[subscription?.currentPlanId ?? 'free']
  const requiredPlanOrder = PLAN_ORDER[minPlan]

  if (currentPlanOrder >= requiredPlanOrder) return <>{children}</>

  return (
    <TouchableOpacity
      onPress={() => navigation.navigate('Subscription' as never)}
      style={styles.lockedWrapper}
    >
      {children}
      <View style={styles.lockOverlay}>
        <Text style={styles.lockText}>🔒 {minPlan === 'standard' ? 'Standard' : 'Premium'}</Text>
        <Text style={styles.lockSub}>Appuyez pour voir les plans</Text>
      </View>
    </TouchableOpacity>
  )
}
```

### Academia gating (courses 3+)
```typescript
// In AcademiaScreen, when mapping courses:
{courses.map((course, index) => (
  index < 2 ? (
    <CourseCard key={course.id} course={course} />
  ) : (
    <PlanGate key={course.id} minPlan="standard" featureName={course.title}>
      <CourseCard course={course} />
    </PlanGate>
  )
))}
```

### Financing amount gating
```typescript
// In ApplyForFinancingSheet, when farmer enters amount:
const getMinPlanForAmount = (amountCdf: number): 'free' | 'standard' | 'premium' | null => {
  if (amountCdf <= 250_000) return null // free tier
  if (amountCdf <= 1_000_000) return 'standard'
  return 'premium'
}

const requiredPlan = getMinPlanForAmount(amount)
if (requiredPlan && PLAN_ORDER[currentPlanId] < PLAN_ORDER[requiredPlan]) {
  // Show upgrade prompt before allowing submit
}
```

## Acceptance criteria
- [ ] Academia shows 2 unlocked courses, remaining courses have lock overlay (free plan)
- [ ] Tapping locked course navigates to SubscriptionScreen
- [ ] Price trend chart shows lock overlay for free plan farmers
- [ ] "Pour vous" buyer section shows lock overlay for free plan farmers
- [ ] Financing form blocks submit when amount exceeds plan limit with upgrade prompt
- [ ] Standard plan users can access all Standard features without lock overlay
- [ ] Premium plan users see no lock overlays anywhere

## Smoke test
1. Sign in as free-plan farmer → open Academia → confirm only first 2 courses accessible
2. Tap a locked course → navigate to SubscriptionScreen
3. Upgrade to Standard (test payment) → return to Academia → all courses accessible
4. Open financing form with free plan → enter 500 000 FC → confirm upgrade prompt appears
