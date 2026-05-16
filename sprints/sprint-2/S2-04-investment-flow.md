# S2-04 — Marketplace — Investment Flow (Firestore Transaction)

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-04 |
| Branch | `feature/s2-04-investment-flow` |
| Owner | Afrotouch OU |
| Estimate | 3.5h |
| Dependencies | S2-03 |

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
git add -A && git commit -m "feat(S2-04): Marketplace — Investment Flow (Firestore Transaction)" && git push origin feature/s2-04-investment-flow
```
