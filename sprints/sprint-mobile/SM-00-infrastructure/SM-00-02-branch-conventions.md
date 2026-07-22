# SM-00-02 — Branch + PR conventions doc

**Sprint:** SM-00 · Infrastructure  
**Branch:** `feature/sm-00-infrastructure`

## Context
The web and admin repos have documented PR conventions in CLAUDE.md. The mobile repo needs the same so the team follows a consistent workflow.

## Acceptance criteria
- [ ] `CLAUDE.md` (or `CONTRIBUTING.md`) added to repo root with:
  - Branch naming: `feature/sm-NN-slug`, `feature/smp-NN-slug` (patch)
  - PR targets `dev`; `dev → main` requires review + CI green
  - Commit message format (conventional commits)
  - Sprint doc DONE rename convention
  - EAS build profile summary (dev / preview / production)
- [ ] `ci.yml` `on.push.branches` includes `dev` (already done) and PR against `main` or `dev`

## Notes
No code changes required — documentation and CI config only.
