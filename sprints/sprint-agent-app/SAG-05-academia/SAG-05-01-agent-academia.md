# SAG-05-01 — Agent Academia (Agent App)

## Context
Agents are field staff who need ongoing training: understanding agricultural cycles, evaluating credit risk, completing regulatory compliance modules (required by Mombongo HQ). Academia on mobile already exists for farmers (SFA-04-01). For agents, the same `getAcademiaModules` CF is called but the CF filters by `role === 'agent'` to return agent-specific courses (not farmer crop courses).

Agent-specific course tracks:
- **Évaluation crédit** — how to score a financing application (required before any assessment)
- **Visite terrain** — field visit best practices, photo documentation
- **Réglementation agricole** — DRC agricultural regulations, land title verification
- **Logistique** — warehouse receipts, delivery confirmation

Agents have no subscription gating — all agent courses are freely accessible. XP and streaks still apply (same CFs).

## Scope
- Add Academia tab to Agent App bottom tab bar
- Reuse `AcademiaScreen.tsx` and `CourseDetailScreen.tsx` — they already handle role-specific courses via the CF's server-side filtering
- Reuse `useAcademiaModules`, `useCompleteModule`, `useAcademiaStreak` hooks (SFA-04-01 and SFA-04-02)
- Add `AgentComplianceBadge` component — shows which required compliance modules are complete; blocks `submitAgentAssessment` if `credit_evaluation` course not completed
- No `PlanGate` — agents don't have a subscription

## Files to create / modify
- `src/navigation/AgentTabNavigator.tsx` — add Academia tab
- `src/components/AgentComplianceBadge.tsx` — compliance status
- `src/screens/pipeline/AssessmentSheet.tsx` — add compliance gate (SAG-03-02)

## Implementation

### AgentTabNavigator.tsx — add tab
```typescript
const tabs = [
  { name: 'Home',          screen: AgentHomeScreen,          icon: 'home' },
  { name: 'Farmers',       screen: AgentFarmerListScreen,    icon: 'account-group' },
  { name: 'Rapports',      screen: AgentReportHistoryScreen, icon: 'clipboard-text' },
  { name: 'Pipeline',      screen: AgentPipelineScreen,      icon: 'kanban' },
  { name: 'Academia',      screen: AcademiaScreen,           icon: 'school' },      // ← new
  { name: 'Notifications', screen: NotificationsScreen,      icon: 'bell' },
]
```

### AgentComplianceBadge.tsx
```typescript
const REQUIRED_COURSES = ['credit_evaluation'] // courseId

export function AgentComplianceBadge() {
  const { data: modules } = useAcademiaModules()

  const requiredCompleted = REQUIRED_COURSES.every(courseId =>
    modules?.find(m => m.courseId === courseId)?.completionPercent === 100
  )

  if (requiredCompleted) return null

  return (
    <View style={styles.badge}>
      <Text style={styles.text}>
        ⚠️ Complétez le cours "Évaluation crédit" avant de soumettre une évaluation
      </Text>
      <TouchableOpacity onPress={() => navigation.navigate('Academia' as never)}>
        <Text style={styles.link}>Aller à l'Academia →</Text>
      </TouchableOpacity>
    </View>
  )
}
```

### AssessmentSheet.tsx — compliance gate
```typescript
import { AgentComplianceBadge } from '@/components/AgentComplianceBadge'
import { useAcademiaModules } from '@/hooks/useAcademia'

// At top of assessment sheet, before the form fields:
<AgentComplianceBadge />

// Disable submit button if required courses not complete:
const { data: modules } = useAcademiaModules()
const creditCourseComplete = modules?.find(m => m.courseId === 'credit_evaluation')?.completionPercent === 100
<Button disabled={!creditCourseComplete} onPress={handleSubmit}>
  Soumettre l'évaluation
</Button>
```

### XP and streak for agents — same hooks, no changes
```typescript
// Agents earn XP and maintain streaks just like farmers.
// The streak reminder push (from SU-02-05 CF) will also fire for agents —
// tap navigates to Academia tab via the agent-specific screen route handler in SAG-00-03.
```

## Cloud Function changes
The `getAcademiaModules` CF must filter returned courses by caller's role:
```typescript
// In getAcademiaModules CF — add role-based filtering:
const role = context.auth?.token?.role ?? 'farmer'
const coursesQuery = db.collection('academiaCourses').where('targetRoles', 'array-contains', role)
```

Each course document in `academiaCourses` collection must have a `targetRoles` array field: `['farmer']`, `['agent']`, or `['farmer', 'agent']` for shared courses.

## ⚠️ Manual step required (Teddy)
Update existing `academiaCourses` Firestore documents to add `targetRoles` field. Existing farmer courses: `targetRoles: ['farmer']`. New agent courses to create: `credit_evaluation`, `field_visit`, `regulation`, `logistics` with `targetRoles: ['agent']`.

## Acceptance criteria
- [ ] Academia tab visible in Agent App bottom bar
- [ ] Agent courses are different from farmer courses (role filtering works)
- [ ] XP badge shown in Academia header for agents
- [ ] Streak widget works for agents (same as SFA-04-02)
- [ ] `AgentComplianceBadge` shown in AssessmentSheet when `credit_evaluation` not completed
- [ ] Assessment submit button disabled until `credit_evaluation` course is 100% complete
- [ ] Completing `credit_evaluation` course enables the submit button

## Smoke test
1. Open Agent App → tap Academia tab → confirm agent-specific courses appear (not farmer crop courses)
2. Open pipeline → tap "Valider" on a farmer → confirm compliance badge shown if credit course incomplete
3. Complete `credit_evaluation` course → return to pipeline → confirm submit button enabled
4. Submit assessment → confirm no compliance error
5. Verify XP badge updates after completing any agent course
