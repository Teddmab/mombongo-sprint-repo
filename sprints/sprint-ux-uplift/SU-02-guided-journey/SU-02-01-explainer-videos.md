# SU-02-01 — VideoExplainer component + first 4 videos

**Sprint:** SU-02 · Guided journey  
**Branch:** `feature/su-02-guided-journey`  
**Effort:** ~8 days (3 dev + 5 video production)

## Context
Short explainer videos (30–60 sec, 2–4 MB each) placed contextually at friction points — never autoplaying, always on user request. This story builds the frontend component and integrates the first 4 highest-priority videos.

## VideoExplainer component (`src/components/VideoExplainer.tsx`)

```tsx
<VideoExplainer slug="onboarding-intro" label="Voir comment ça marche →" />
```

Behavior:
1. Renders as a compact "▶ Voir comment ça marche" button
2. On tap: calls `httpsCallable(functions, 'getVideoUrl')({ slug })` → receives signed URL (24h)
3. Shows a bottom sheet (mobile) or inline player (desktop) with `<video controls>` 
4. Falls back gracefully if CF fails: shows text summary instead
5. Tracks play event via Firebase Analytics `logEvent('video_play', { slug })`
6. Respects `html[data-slow]`: on slow connections, shows "Téléchargez cette vidéo en WiFi" instead of streaming

### CF: `getVideoUrl(slug: string)` → `{ url: string, title: string, durationSec: number }`
- Reads video metadata from `config/videos/{slug}` document
- Returns Firebase Storage signed URL (24h expiry)
- Frontend never holds a direct Storage URL

## Video placement for Sprint 2

| Slug | Placement | Screen | Duration |
|------|-----------|--------|----------|
| `onboarding-intro` | Step 4 of onboarding flow | OnboardingScreen | 60 sec |
| `create-exploitation` | Empty state of Exploitation screen | ExploitationScreen | 30 sec |
| `bourse-how-to-sell` | Empty state of Bourse (no listings yet) | BourseScreen | 45 sec |
| `financement-how-credit-works` | Step 1 of Financement wizard | FinancementScreen | 45 sec |

## Video production spec (for content team)
- Resolution: 854×480 (480p), MP4 H.264
- Bitrate: max 800 kbps → approx 2–4 MB for 30–60 sec
- Audio: French voiceover, clear, no background music (compression artifact risk)
- Subtitles: `.vtt` file in Lingala, stored alongside video
- Content: screen recordings of the app + voiceover describing each action
- Delivery: upload to Firebase Storage at `videos/{slug}/video.mp4` and `videos/{slug}/subtitles-ln.vtt`

## Data model
```
config/videos/{slug}/
  title: string
  durationSec: number
  storagePath: string       ← e.g. "videos/onboarding-intro/video.mp4"
  subtitlesPath: string     ← e.g. "videos/onboarding-intro/subtitles-ln.vtt"
  sizeBytes: number
  active: boolean
```

## Acceptance criteria
- [ ] `VideoExplainer` component renders as a compact button (not autoplaying)
- [ ] Tapping the button fetches a signed URL and opens the player
- [ ] Player shows captions/subtitles button when `.vtt` is available
- [ ] On `html[data-slow]`: button label changes to "Disponible en WiFi" and is disabled
- [ ] CF `getVideoUrl` returns 404-equivalent if slug doesn't exist (handled gracefully)
- [ ] All 4 video placements render without error in dev mode (mock URL returned)
- [ ] Analytics event `video_play` fires when video starts

## Smoke test steps
1. Open Exploitation screen with no exploitations → verify "▶ Voir comment" button visible
2. Tap button → verify player opens, video plays, captions available
3. Enable Network Throttle → slow 3G → tap button → verify "Disponible en WiFi" state
4. Check Firebase Console → Analytics → verify `video_play` event recorded
5. Open Financement wizard → verify video button on step 1
6. Open Bourse with no listings → verify video button in empty state
