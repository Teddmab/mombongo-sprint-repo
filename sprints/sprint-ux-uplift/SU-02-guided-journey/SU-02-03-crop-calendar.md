# SU-02-03 — Crop calendar on Exploitation screen

**Sprint:** SU-02 · Guided journey  
**Branch:** `feature/su-02-guided-journey`  
**Effort:** ~6 days (3 CF + 2 frontend + 1 test)

## Context
The exploitation screen shows a static record of the farm. This story adds a "Calendrier cultural" section that displays the current phase of the farmer's crop cycle and the next 3 upcoming events, based on planting date and crop type. This creates a reason to check the app weekly without any additional input from the farmer.

## Crop phase model

Phase timeline per crop (approximate — can be refined per region):

| Phase | Maïs | Manioc | Arachide | Soja |
|-------|------|--------|----------|------|
| Semis | Day 0 | Day 0 | Day 0 | Day 0 |
| Germination | Days 5–10 | Days 7–14 | Days 3–5 | Days 4–6 |
| Croissance végétative | Days 10–45 | Days 14–120 | Days 5–35 | Days 6–35 |
| Floraison | Days 45–65 | N/A | Days 35–50 | Days 35–55 |
| Grain/Tubercule | Days 65–90 | Days 120–240 | Days 50–80 | Days 55–90 |
| Maturité | Days 90–110 | Days 240–300 | Days 80–100 | Days 90–120 |
| Récolte | Days 110–120 | Days 300–360 | Days 100–110 | Days 120–130 |

Corresponding recommended actions per phase:
- Germination: "Vérifiez la levée — 90% minimum requis"
- Croissance végétative: "Désherbage recommandé · Fertilisation si sol analysé"
- Floraison: "Évitez les pesticides — risque de pollinisation compromise"
- Grain/Tubercule: "Traitement contre les ravageurs si nécessaire"
- Maturité: "Préparez votre stockage · Contactez des acheteurs sur la Bourse"
- Récolte: "Récolte optimale cette semaine · Publiez votre annonce maintenant"

## Implementation

### CF: `getCropCalendar(exploitationId)` 
Returns:
```typescript
{
  currentPhase: { name: string, daysIn: number, totalDays: number },
  nextEvents: Array<{ date: string, label: string, urgency: 'info' | 'action' | 'urgent' }>,
  harvestEstimate: string,   // "dans 47 jours" or "cette semaine"
  percentComplete: number    // 0–100
}
```

Logic: reads `exploitations/{id}.plantingDate` and `primaryCrop`, computes phase from `daysSincePlanting`.

### New hook: `useCropCalendar(exploitationId)` (`src/hooks/useCropCalendar.ts`)
- `isDevMode()` → returns mock calendar for Maïs, day 62 (floraison phase)

### UI: `CropCalendarWidget` on `ExploitationScreen` / exploitation detail view
- Progress bar: "Phase : Floraison · Jour 62 / 90"  
- Horizontal phases strip (mini timeline) highlighting current phase
- "Prochains événements" list: 3 upcoming events with dates and urgency colors
- "Récolte estimée dans 47 jours" badge
- If `plantingDate` not set: "Ajoutez la date de semis pour activer le calendrier →"

### Push alerts (uses SU-01-05 FCM infrastructure)
- CF `checkCropAlerts` runs daily, sends push when an event is within 3 days
- Example: "Votre Maïs entre en phase de floraison dans 2 jours — évitez les pesticides"

## Acceptance criteria
- [ ] CropCalendarWidget renders on the exploitation detail screen
- [ ] Current phase shown with days in / total days
- [ ] Harvest estimate date shown ("dans X jours" or "cette semaine")
- [ ] If no plantingDate: prompt to add it, widget not shown
- [ ] 3 upcoming events listed with urgency colors (green/amber/red)
- [ ] Horizontal phase strip highlights current phase correctly
- [ ] Push alert fires 3 days before a phase transition (smoke: manual CF trigger)
- [ ] `isDevMode()` returns Maïs day-62 mock without CF call

## Smoke test steps
1. Open an exploitation with plantingDate set → verify CropCalendarWidget renders
2. Verify current phase label and progress bar match expected day count
3. Verify "Prochain événement" has a date within the next 2 weeks
4. Set plantingDate to near-harvest date → verify "Récolte cette semaine" badge
5. Open exploitation with no plantingDate → verify "Ajoutez la date de semis" prompt
6. Manually trigger `checkCropAlerts` CF → verify push received on test device
