# S7-01 — PWA Manifest and Offline Mode

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S7-01 |
| Branch | `feature/s7-01-pwa-offline` |
| Owner | Afrotouch OU |
| Estimate | 2h |
| Dependencies | S0-01 |

---

## Context
Full Lovable prompt, unit tests, regression gates and PR checklist are in PLAYBOOK.md.
Follow PLAYBOOK.md step by step for this story.

---

## Quick Regression Gate
```bash
bun run typecheck && bun run lint && bun run test:unit && bun run build
```

## ✅ Definition of Done
- [ ] Lovable prompt implemented and tested in browser
- [ ] Unit tests written and passing (`bun run test:unit`)
- [ ] `bun run test:ci` exits 0
- [ ] `bun run build` exits 0
- [ ] Zero hardcoded strings — all text uses `t()`
- [ ] PR reviewed and merged to `dev`

```bash
git add -A && git commit -m "feat(S7-01): PWA Manifest and Offline Mode" && git push origin feature/s7-01-pwa-offline
```
