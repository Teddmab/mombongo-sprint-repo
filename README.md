# Mombongo — Sprint Repository

**Afrotouch OU × Nodematics LTD**

This repo is the single source of truth for the Mombongo platform build.
It contains sprint story files, CI/CD workflows, and the master playbook.

---

## Repo map

```
mombongo-sprint-repo/          ← YOU ARE HERE
├── README.md
├── PLAYBOOK.md                ← master step-by-step guide
├── .github/
│   └── workflows/
│       ├── ci.yml             ← runs on every PR to dev/staging/main
│       ├── staging.yml        ← deploys on merge to staging
│       └── production.yml     ← deploys on merge to main (manual approval)
└── sprints/
    ├── sprint-0/              ← Infrastructure (Day Off 1)
    │   ├── S0-01-repo-setup.md
    │   ├── S0-02-firebase-setup.md
    │   ├── S0-03-design-system.md
    │   └── S0-04-auth-routing.md
    ├── sprint-1/              ← Auth & Home (Day Off 2)
    │   ├── S1-01-language-screen.md
    │   ├── S1-02-auth-screen.md
    │   └── S1-03-home-screen.md
    ├── sprint-2/              ← Marketplace (Day Off 3)
    │   ├── S2-01-product-list.md
    │   ├── S2-02-filter-sheet.md
    │   ├── S2-03-product-detail.md
    │   ├── S2-04-investment-flow.md
    │   └── S2-05-portfolio.md
    ├── sprint-3/              ← Bourse (Week 2)
    │   ├── S3-01-bourse-screen.md
    │   ├── S3-02-bourse-detail.md
    │   └── S3-03-bourse-invest.md
    ├── sprint-4/              ← Farmer Financing (Week 2)
    │   ├── S4-01-financing-screen.md
    │   ├── S4-02-farmer-detail.md
    │   ├── S4-03-agent-report.md
    │   └── S4-04-cultural-calendar.md
    ├── sprint-5/              ← Academia (Week 3)
    │   ├── S5-01-academia-screen.md
    │   ├── S5-02-course-detail.md
    │   └── S5-03-module-player.md
    ├── sprint-6/              ← Admin Dashboard (Week 3)
    │   ├── S6-01-admin-dashboard.md
    │   ├── S6-02-admin-users.md
    │   ├── S6-03-admin-financing.md
    │   └── S6-04-admin-bourse.md
    └── sprint-7/              ← PWA & Production (Week 4)
        ├── S7-01-pwa-offline.md
        ├── S7-02-fcm-notifications.md
        ├── S7-03-cinetpay-integration.md
        ├── S7-04-polish-i18n.md
        └── S7-05-production-deploy.md
```

---

## Application repos

| Repo | Role | Stack |
|------|------|-------|
| `mombongo-web` | Frontend PWA | React + Vite + Firebase (Lovable) |
| `mombongo-backoffice` | Patrick / agent tool | React + Vite + Firebase (Lovable) |
| `mombongo-admin` | djuna sambil panel | React + Vite + Firebase (Lovable) |
| `mombongo-mobile` | iOS + Android | React Native + Expo |
| `mombongo-functions` | Backend | Firebase Cloud Functions |
| `mombongo-sprint-repo` | This repo | Markdown + CI/CD |

---

## Branch strategy (applies to mombongo-web, backoffice, admin)

```
feature/sN-NN-slug  →  dev  →  staging  →  main
                       (CI)    (CI+E2E)    (approval+smoke)
```

## Test commands (run inside mombongo-web)

```bash
bun run dev                   # local dev server
bun run typecheck             # TypeScript check
bun run lint                  # ESLint
bun run test:unit             # Vitest (Firebase mocked)
bun run test:unit:watch       # Vitest watch mode
bun run emulator:start        # Firebase Emulator
bun run test:integration      # Vitest + real emulator
bun run test:e2e              # Playwright
bun run test:ci               # unit + integration (CI gate)
bun run build                 # production build
```

---

*Owner: Patrick Kadima — Nodematics LTD | Tech Lead: Afrotouch OU*
