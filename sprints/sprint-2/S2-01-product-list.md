# S2-01 — Marketplace — Product List and Search

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-01 |
| Branch | `feature/s2-01-product-list` |
| Owner | Afrotouch OU |
| Estimate | 2.5h |
| Dependencies | S0-02 S0-03 S0-04 |

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
git add -A && git commit -m "feat(S2-01): Marketplace — Product List and Search" && git push origin feature/s2-01-product-list
```
