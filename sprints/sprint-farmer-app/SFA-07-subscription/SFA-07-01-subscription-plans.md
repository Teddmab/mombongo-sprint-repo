# SFA-07-01 — Subscription Plans (Farmer App)

## Context
Port of SM-07-00. Mombongo plans to offer tiered subscriptions (Gratuit / Standard / Premium) that unlock features and increase financing limits. The web has `SubscriptionModal.tsx` but it's not wired to real plans or payment CFs. This sprint creates the real subscription plan UI and wires it to the `getSubscriptionPlans` and `subscribeToplan` CFs.

**Payment provider:** PawaPay for mobile money, Stripe for card. Same `initiatePayment` CF as SFA-03-03.

## Subscription tiers
| Tier | Price | Financing limit | Key perks |
|------|-------|-----------------|-----------|
| Gratuit | 0 FC/mo | 250 000 FC | Basic market prices, 2 Academia courses |
| Standard | 2 500 FC/mo | 1 000 000 FC | All market prices, all Academia, priority support |
| Premium | 7 500 FC/mo | 5 000 000 FC | All Standard + Bourse priority listing, dedicated agent |

## Scope
- Create `src/hooks/useSubscription.ts` — calls `getSubscriptionPlans` + `subscribeToplan` CFs
- Create `src/screens/subscription/SubscriptionScreen.tsx`
- Create `src/components/PlanCard.tsx`
- Add "Mon abonnement" row in ProfileScreen → navigates to SubscriptionScreen
- Current plan badge shown in ProfileScreen

## Cloud Functions required
- `getSubscriptionPlans` — returns plan definitions + farmer's current plan
- `subscribeToPlan` — input: `{ planId, paymentMethod }` → initiates payment + activates plan on success

## Files to create
- `src/hooks/useSubscription.ts`
- `src/screens/subscription/SubscriptionScreen.tsx`
- `src/components/PlanCard.tsx`

## Implementation

### `src/hooks/useSubscription.ts`
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'
import { isDevMode } from '@/lib/utils'

export type Plan = {
  id: 'free' | 'standard' | 'premium'
  name: string
  priceCdf: number
  financingLimitCdf: number
  features: string[]
}

export type SubscriptionData = {
  currentPlanId: 'free' | 'standard' | 'premium'
  renewalDate?: string
  plans: Plan[]
}

const MOCK_SUBSCRIPTION: SubscriptionData = {
  currentPlanId: 'free',
  plans: [
    { id: 'free', name: 'Gratuit', priceCdf: 0, financingLimitCdf: 250_000, features: ['Prix marché basiques', '2 cours Academia'] },
    { id: 'standard', name: 'Standard', priceCdf: 2_500, financingLimitCdf: 1_000_000, features: ['Tous les prix marché', 'Toute l\'Academia', 'Support prioritaire'] },
    { id: 'premium', name: 'Premium', priceCdf: 7_500, financingLimitCdf: 5_000_000, features: ['Tout Standard', 'Publication Bourse prioritaire', 'Agent dédié'] },
  ],
}

export function useSubscription() {
  return useQuery({
    queryKey: ['subscription'],
    queryFn: async (): Promise<SubscriptionData> => {
      if (isDevMode()) return MOCK_SUBSCRIPTION
      const res = await httpsCallable<void, SubscriptionData>(functions, 'getSubscriptionPlans')()
      return res.data
    },
  })
}

export function useSubscribeToPlan() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (payload: { planId: string; paymentMethod: 'pawapay' | 'card' }) =>
      httpsCallable(functions, 'subscribeToPlan')(payload),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['subscription'] }),
  })
}
```

### SubscriptionScreen.tsx
```typescript
// Header: current plan badge + renewal date
// Plan cards in vertical list (free → standard → premium)
// "Votre plan actuel" badge on current plan
// "Passer à [plan]" button on upgradeable plans → opens PaymentModal with subscription amount
// Downgrade: "Revenir au plan Gratuit" (effective at end of billing period)
```

### PlanCard.tsx
```typescript
export function PlanCard({ plan, isCurrent, onUpgrade }: { plan: Plan; isCurrent: boolean; onUpgrade: () => void }) {
  return (
    <View style={[styles.card, isCurrent && styles.currentCard]}>
      {isCurrent && <View style={styles.currentBadge}><Text>Plan actuel</Text></View>}
      <Text style={styles.name}>{plan.name}</Text>
      <Text style={styles.price}>
        {plan.priceCdf === 0 ? 'Gratuit' : `${plan.priceCdf.toLocaleString()} FC/mois`}
      </Text>
      <Text style={styles.limit}>Financement jusqu'à {(plan.financingLimitCdf / 1000).toFixed(0)}K FC</Text>
      {plan.features.map(f => <Text key={f} style={styles.feature}>✓ {f}</Text>)}
      {!isCurrent && plan.priceCdf > 0 && (
        <TouchableOpacity style={styles.upgradeBtn} onPress={onUpgrade}>
          <Text>Passer à {plan.name}</Text>
        </TouchableOpacity>
      )}
    </View>
  )
}
```

## Acceptance criteria
- [ ] Subscription screen shows 3 plan cards with current plan highlighted
- [ ] Tapping "Passer à Standard" opens PaymentModal with correct amount (2 500 FC)
- [ ] After payment success, current plan updates to Standard
- [ ] Renewal date shown for paid plans
- [ ] ProfileScreen shows current plan badge

## Smoke test
1. Open Profile → "Mon abonnement" → 3 plan cards visible
2. Current plan shows "Plan actuel" badge (Free initially)
3. Tap "Passer à Standard" → PaymentModal opens with 2 500 FC
4. Complete PawaPay test payment → confirm current plan updates to Standard
5. Reload app → Standard plan still shown (persisted)
