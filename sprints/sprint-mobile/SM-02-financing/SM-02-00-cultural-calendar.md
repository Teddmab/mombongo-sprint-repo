# SM-02-00 — Cultural calendar in financing screens

**Sprint:** SM-02 · Financing  
**Branch:** `feature/sm-02-financing`

## Context
The web app's financing tab has a "Calendrier cultural" tab showing crop planting/harvest dates per region. Mobile financing screens have no calendar view — users cannot track their agricultural cycle.

## Acceptance criteria
- [ ] `FarmerFinancementScreen` has a second tab "Calendrier" alongside "Mes financements"
- [ ] Calendar shows a scrollable month view with planting and harvest markers
- [ ] Each entry shows: crop name, date, action type (Plantation / Récolte / Traitement)
- [ ] `InvestorFinancementScreen` shows calendar of portfolio farmers' harvest dates
- [ ] Data from: `httpsCallable(functions, "getCulturalCalendar")` or derived from financing data
- [ ] In devMode, returns mock calendar entries from `data/mock.ts`
- [ ] `useCulturalCalendar(farmerId?)` hook added to `hooks/useFinancing.ts`

## Data shape
```ts
interface CalendarEntry {
  id: string;
  farmerId: string;
  farmerName: string;
  cropType: string;
  eventType: "planting" | "harvest" | "treatment";
  date: { seconds: number };
  note?: string;
}
```

## Implementation notes
- UI: vertical list of months with day markers (no complex calendar grid needed)
- Color coding: green = planting, amber = treatment, blue = harvest
- Web i18n key: `financing.calendar.title`, `financing.calendar.crop`, etc. — mirror in mobile i18n
