# SFA-02-04 — Explainer Videos (Farmer App)

## Context
Port of web sprint SU-02-01 to mobile. Short educational videos (60–120s, 4G-optimized, 3G-friendly thumbnails) that explain key platform actions in context — not a standalone "video library" but a contextual nudge: the right video appears on the screen where the farmer might be confused.

Web uses a `VideoExplainer` component that calls `getVideoUrl` CF (returns a signed GCS URL). Mobile uses `expo-av` (`Video` component) for playback. DRC connectivity constraint: videos must not auto-play; farmer explicitly taps to load.

Placement per screen:
- **Bourse publish** → "Comment publier votre récolte" (90s)
- **Financement apply** → "Comment faire une demande de financement" (120s)
- **Exploitation create** → "Comment enregistrer votre exploitation" (60s)
- **Academia home** → "Comment utiliser l'Academia" (60s)

## Scope
- Create `src/components/VideoExplainerCard.tsx` — tap-to-load with thumbnail, uses `expo-av`
- Create `src/hooks/useVideoUrl.ts` — calls `getVideoUrl` CF (returns signed GCS URL)
- Add `VideoExplainerCard` to the 4 screens above
- Thumbnail shown always (low-res image from GCS); video loads only on tap
- "Ne plus afficher" option persists to AsyncStorage per video ID

## Cloud Function required
`getVideoUrl` — planned in SU-02-01 (web). Same CF reused on mobile. Input: `{ videoId: string }` → output: `{ url: string; thumbnailUrl: string; durationSeconds: number }`

## Files to create
- `src/hooks/useVideoUrl.ts`
- `src/components/VideoExplainerCard.tsx`

## Implementation

### `src/hooks/useVideoUrl.ts`
```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions } from '@/lib/firebase'

const VIDEO_IDS = {
  bourse_publish: 'video_bourse_publish',
  financement_apply: 'video_financement_apply',
  exploitation_create: 'video_exploitation_create',
  academia_intro: 'video_academia_intro',
} as const

export type VideoKey = keyof typeof VIDEO_IDS

export function useVideoUrl(key: VideoKey, enabled: boolean) {
  return useQuery({
    queryKey: ['videoUrl', key],
    queryFn: async () => {
      const res = await httpsCallable<{ videoId: string }, { url: string; thumbnailUrl: string; durationSeconds: number }>(
        functions, 'getVideoUrl'
      )({ videoId: VIDEO_IDS[key] })
      return res.data
    },
    enabled,
    staleTime: 60 * 60_000, // signed URLs valid for 1h
  })
}
```

### VideoExplainerCard.tsx
```typescript
import { Video, ResizeMode } from 'expo-av'
import AsyncStorage from '@react-native-async-storage/async-storage'

export function VideoExplainerCard({ videoKey, title }: { videoKey: VideoKey; title: string }) {
  const [dismissed, setDismissed] = useState<boolean | null>(null)
  const [playing, setPlaying] = useState(false)
  const { data: video, refetch } = useVideoUrl(videoKey, playing)

  useEffect(() => {
    AsyncStorage.getItem(`video_dismissed_${videoKey}`).then(v => setDismissed(v === 'true'))
  }, [videoKey])

  if (dismissed === null || dismissed) return null

  return (
    <View style={styles.card}>
      <View style={styles.header}>
        <Text style={styles.title}>▶ {title}</Text>
        <TouchableOpacity onPress={async () => {
          await AsyncStorage.setItem(`video_dismissed_${videoKey}`, 'true')
          setDismissed(true)
        }}>
          <Text style={styles.dismiss}>Ne plus afficher</Text>
        </TouchableOpacity>
      </View>

      {!playing ? (
        <TouchableOpacity style={styles.thumbnail} onPress={() => { setPlaying(true); refetch() }}>
          {/* Thumbnail image from thumbnailUrl */}
          <View style={styles.playButton}>
            <Text style={styles.playIcon}>▶</Text>
          </View>
          {video && <Text style={styles.duration}>{Math.floor(video.durationSeconds / 60)}:{String(video.durationSeconds % 60).padStart(2, '0')}</Text>}
        </TouchableOpacity>
      ) : video ? (
        <Video
          source={{ uri: video.url }}
          style={styles.video}
          useNativeControls
          resizeMode={ResizeMode.CONTAIN}
          shouldPlay
        />
      ) : (
        <ActivityIndicator />
      )}
    </View>
  )
}
```

### Usage in screens
```typescript
// In FarmerBourseScreen above the listings list:
<VideoExplainerCard videoKey="bourse_publish" title="Comment publier votre récolte" />

// In FarmerFinancementScreen:
<VideoExplainerCard videoKey="financement_apply" title="Comment faire une demande" />
```

## Install command
```bash
npx expo install expo-av
```

## Acceptance criteria
- [ ] Video explainer card appears in Bourse, Financement, Exploitation, and Academia screens
- [ ] Thumbnail visible immediately (no CF call until tap)
- [ ] Tapping thumbnail loads and plays video via signed URL
- [ ] "Ne plus afficher" hides the card permanently for that video (persists across restarts)
- [ ] Card not shown for users who dismissed it previously
- [ ] Video does not auto-play (farmer must tap)

## Smoke test
1. Open Bourse tab (fresh user) → confirm video card appears with thumbnail
2. Tap thumbnail → confirm video loads and plays
3. Tap "Ne plus afficher" → card disappears
4. Kill app + reopen → confirm video card still gone
5. Open Financement tab → confirm its video card is independent (still shown)
