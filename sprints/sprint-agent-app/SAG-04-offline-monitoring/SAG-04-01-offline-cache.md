# SAG-04-01 — Offline Cache (Agent App)

## Context
Same setup as SFA-05-03 (persisted React Query + AsyncStorage). For agents, the offline use case is even more critical: agents visit farms in areas with no connectivity. They need their farmer list and visit plan to be available offline.

Key difference from the farmer app: agents need to be able to **submit a draft report** while offline and have it sync when connectivity returns. This requires an optimistic queue, not just a read cache.

## Scope
- Same persisted QueryClient setup as SFA-05-03
- Add `reportDraftQueue` in AsyncStorage — store incomplete reports for later sync
- When connectivity restores, drain the queue (call `submitAgentReport` for each queued draft)
- OfflineBanner same as SFA-05-03

## Files to create / modify
- `App.tsx` — persisted QueryClient (shared setup with farmer app)
- `src/lib/reportQueue.ts` — draft queue management
- `src/screens/reports/AgentReportFormScreen.tsx` — add "Enregistrer brouillon" that queues when offline

## Implementation

### `src/lib/reportQueue.ts`
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage'

const QUEUE_KEY = 'agent_report_draft_queue'

type DraftReport = {
  id: string
  payload: ReportPayload
  createdAt: string
  synced: boolean
}

export async function enqueueDraft(payload: ReportPayload): Promise<void> {
  const raw = await AsyncStorage.getItem(QUEUE_KEY)
  const queue: DraftReport[] = raw ? JSON.parse(raw) : []
  queue.push({ id: Date.now().toString(), payload, createdAt: new Date().toISOString(), synced: false })
  await AsyncStorage.setItem(QUEUE_KEY, JSON.stringify(queue))
}

export async function getDrafts(): Promise<DraftReport[]> {
  const raw = await AsyncStorage.getItem(QUEUE_KEY)
  return raw ? JSON.parse(raw) : []
}

export async function removeDraft(id: string): Promise<void> {
  const raw = await AsyncStorage.getItem(QUEUE_KEY)
  const queue: DraftReport[] = raw ? JSON.parse(raw) : []
  await AsyncStorage.setItem(QUEUE_KEY, JSON.stringify(queue.filter(d => d.id !== id)))
}
```

### Queue drain in AuthContext / App startup
```typescript
import NetInfo from '@react-native-community/netinfo'
import { getDrafts, removeDraft } from '@/lib/reportQueue'

// On connectivity restore:
NetInfo.addEventListener(async state => {
  if (state.isConnected) {
    const drafts = await getDrafts()
    for (const draft of drafts) {
      try {
        await httpsCallable(functions, 'submitAgentReport')(draft.payload)
        await removeDraft(draft.id)
      } catch {
        // Leave in queue — will retry on next reconnect
      }
    }
  }
})
```

## Acceptance criteria
- [ ] Agent farmer list available offline (from cache)
- [ ] Visit plan available offline (from cache)
- [ ] "Enregistrer brouillon" saves report to local queue when offline
- [ ] On reconnection, draft reports automatically submit to CF
- [ ] Submitted drafts removed from queue; failed ones kept for retry
- [ ] OfflineBanner shows when offline

## Smoke test
1. Load farmer list + visit plan with internet, then disable wifi
2. Open app again — confirm farmer list and visit plan show from cache
3. Write a report → tap "Enregistrer brouillon"
4. Re-enable wifi → confirm report is submitted (check Firestore)
5. Confirm draft no longer in queue (no double submission)
