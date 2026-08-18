# SFA-03-03 — Payments: PawaPay + Stripe Card (Farmer App)

## Context
Port of SM-08. The mobile PaymentModal already has a multi-step flow and PawaPay mobile money. Stripe card payment is currently a stub ("card" method shows nothing). This sprint completes the Stripe card integration and polishes the PawaPay UX.

**Payment provider rule (from memory):** PawaPay for mobile money, Stripe for Visa/Mastercard. No CinetPay.

The web already has both working via `initiatePayment` and `confirmPayment` CFs. We reuse the same CFs on mobile.

## Scope
- Complete Stripe card payment in `PaymentModal` using `@stripe/stripe-react-native`
- Polish PawaPay flow: phone number validation (DRC format), operator auto-detection
- Add `usePaymentStatus` polling hook (check `getPaymentStatus` CF every 5s until resolved)
- Both flows end at same success screen showing transaction ID

## Cloud Functions required (already deployed)
- `initiatePayment` — creates payment intent, returns `{ paymentIntentId, clientSecret }` for Stripe; `{ paymentId, instructionCode }` for PawaPay
- `confirmPayment` — called after PawaPay user completes on their end
- `getPaymentStatus` — poll for PawaPay payment status

## Files to modify
- `src/components/PaymentModal.tsx` — add Stripe card section, fix card method stub
- `src/hooks/usePaymentStatus.ts` — polling hook

## Implementation

### Stripe card in PaymentModal.tsx
```typescript
import { useStripe } from '@stripe/stripe-react-native'

// In the "card" payment method branch:
const { confirmPayment } = useStripe()

const handleCardPayment = async () => {
  const { data } = await httpsCallable<{ amount: number; currency: string }, { clientSecret: string }>(
    functions, 'initiatePayment'
  )({ amount, currency: 'CDF', method: 'card' })

  const { error } = await confirmPayment(data.clientSecret, {
    paymentMethodType: 'Card',
    paymentMethodData: { billingDetails: { /* from form */ } },
  })

  if (error) throw new Error(error.message)
  // success — invalidate wallet query
}
```

### PawaPay polish
```typescript
// Phone number validation: DRC format +243XXXXXXXXX
// Operator auto-detection from prefix:
const detectOperator = (phone: string): 'MPESA' | 'ORANGE' | 'AIRTEL' | null => {
  const cleaned = phone.replace(/\D/g, '')
  if (cleaned.startsWith('243 81') || cleaned.startsWith('243 82')) return 'MPESA'
  if (cleaned.startsWith('243 84') || cleaned.startsWith('243 85')) return 'ORANGE'
  if (cleaned.startsWith('243 97') || cleaned.startsWith('243 99')) return 'AIRTEL'
  return null
}
```

### `src/hooks/usePaymentStatus.ts`
```typescript
export function usePaymentStatus(paymentId: string | null) {
  return useQuery({
    queryKey: ['paymentStatus', paymentId],
    queryFn: async () => {
      const res = await httpsCallable<{ paymentId: string }, { status: string }>(
        functions, 'getPaymentStatus'
      )({ paymentId: paymentId! })
      return res.data.status
    },
    enabled: !!paymentId,
    refetchInterval: (data) => (data === 'pending' ? 5000 : false),
  })
}
```

## Install command
```bash
npx expo install @stripe/stripe-react-native
```

## Acceptance criteria
- [ ] Stripe card flow: entering card details + confirm completes payment successfully
- [ ] PawaPay: entering DRC phone number auto-detects operator (M-Pesa/Orange/Airtel)
- [ ] PawaPay: status polls every 5s until resolved; success screen shown
- [ ] Both flows show transaction ID on success screen
- [ ] Payment failure shows error message (not crash)

## Smoke test
1. Open PaymentModal → select "Carte Visa/Mastercard"
2. Enter Stripe test card 4242 4242 4242 4242 → confirm
3. Verify success screen with transaction ID
4. Open PaymentModal → select "Mobile Money"
5. Enter +243 81X XXX XXXX → verify "M-Pesa" auto-detected
6. Complete payment → verify status polling → success screen
