# SM-07-01 — Premium content gating in Academia

**Sprint:** SM-07 · Subscription  
**Branch:** `feature/sm-07-subscription`

## Context
After subscription is working (SM-07-00), premium courses and modules must be locked for non-subscribers. The gating must be enforced both in the UI (showing a lock) and at the CF level (completeModule checks subscription).

## Acceptance criteria
- [ ] `CourseCard` in `AcademiaScreen` shows 🔒 badge on premium courses for non-subscribers
- [ ] `CourseDetailScreen`: if `course.isPremium && !subscription.isActive`:
  - Module list shows lock icon on non-free modules
  - "S'inscrire" CTA replaced with "Débloquer avec Premium" → opens `SubscriptionModal`
- [ ] `ModulePlayerModal`: if module is not free and user not subscribed, shows paywall screen instead of content
- [ ] `useSubscriptionStatus()` hook available in all academia components
- [ ] Module free indicator: `module.isFree === true` → always accessible regardless of subscription
- [ ] CF `completeModule` validates subscription server-side (returns 403 if premium + not subscribed)

## Implementation notes
- Derive subscription status from AuthContext (add `subscriptionActive: boolean` field)
- "Débloquer" CTA: deeplink to SubscriptionModal via a shared context or navigation
- Free modules (e.g. first module of each course) always accessible — use as a "try before subscribe" hook
