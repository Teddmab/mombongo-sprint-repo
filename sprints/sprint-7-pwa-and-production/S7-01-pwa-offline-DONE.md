# S7-01 — PWA — Manifest, Service Worker & Offline Mode

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S7-01 |
| Sprint | Sprint 7 — PWA & Production |
| Branch | `feature/s7-01-pwa-offline` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S1-03 (OfflineBanner exists) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | vite-plugin-pwa setup, manifest, service worker, install prompt |
| `mombongo-admin` | ✅ Done | — |
| `mombongo-functions` | ✅ Done | — |

---

## mombongo-web

### Step 1 — Install vite-plugin-pwa

```bash
npm install -D vite-plugin-pwa
```

### Step 2 — Configure vite.config.ts

```typescript
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.ico', 'icons/*.png'],
      manifest: {
        name:             'Mombongo',
        short_name:       'Mombongo',
        description:      'Investissez dans l\'agriculture congolaise',
        theme_color:      '#16a34a',
        background_color: '#ffffff',
        display:          'standalone',
        orientation:      'portrait',
        start_url:        '/',
        scope:            '/',
        icons: [
          { src: '/icons/icon-72x72.png',   sizes: '72x72',   type: 'image/png' },
          { src: '/icons/icon-96x96.png',   sizes: '96x96',   type: 'image/png' },
          { src: '/icons/icon-128x128.png', sizes: '128x128', type: 'image/png' },
          { src: '/icons/icon-192x192.png', sizes: '192x192', type: 'image/png', purpose: 'any maskable' },
          { src: '/icons/icon-512x512.png', sizes: '512x512', type: 'image/png', purpose: 'any maskable' },
        ],
      },
      workbox: {
        // Cache app shell + static assets
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            // Cache Firebase API calls for offline read
            urlPattern: /^https:\/\/firestore\.googleapis\.com/,
            handler:    'StaleWhileRevalidate',
            options: {
              cacheName:        'firestore-cache',
              expiration:       { maxEntries: 100, maxAgeSeconds: 86_400 },
              cacheableResponse: { statuses: [0, 200] },
            },
          },
          {
            // Cache product images
            urlPattern: /^https:\/\/storage\.googleapis\.com/,
            handler:    'CacheFirst',
            options: {
              cacheName:  'image-cache',
              expiration: { maxEntries: 50, maxAgeSeconds: 604_800 },
            },
          },
        ],
      },
    }),
  ],
})
```

### Step 3 — App icons

Create placeholder icons in `public/icons/` at the sizes listed in manifest. Generate from the Mombongo logo SVG using a tool like `pwa-asset-generator`:

```bash
npx pwa-asset-generator public/logo.svg public/icons --manifest public/manifest.json --index public/index.html
```

### Step 4 — Install prompt (A2HS)

Create `src/hooks/usePwaInstall.ts`:

```typescript
import { useEffect, useState } from 'react'

interface BeforeInstallPromptEvent extends Event {
  prompt: () => Promise<void>
  userChoice: Promise<{ outcome: 'accepted' | 'dismissed' }>
}

export function usePwaInstall() {
  const [installEvent, setInstallEvent] = useState<BeforeInstallPromptEvent | null>(null)
  const [installed, setInstalled] = useState(false)

  useEffect(() => {
    const handler = (e: Event) => {
      e.preventDefault()
      setInstallEvent(e as BeforeInstallPromptEvent)
    }
    window.addEventListener('beforeinstallprompt', handler)
    window.addEventListener('appinstalled', () => setInstalled(true))
    return () => window.removeEventListener('beforeinstallprompt', handler)
  }, [])

  async function install() {
    if (!installEvent) return
    await installEvent.prompt()
    const { outcome } = await installEvent.userChoice
    if (outcome === 'accepted') setInstalled(true)
    setInstallEvent(null)
  }

  return { canInstall: !!installEvent && !installed, install }
}
```

Add **InstallBanner** to `MobileHome`:

```tsx
const { canInstall, install } = usePwaInstall()

{canInstall && (
  <div
    data-testid="pwa-install-banner"
    className="mx-4 mt-2 p-3 bg-green-50 border border-green-200 rounded-2xl flex items-center gap-3"
  >
    <span className="text-2xl">📲</span>
    <div className="flex-1">
      <p className="font-bold text-sm text-green-800">{t('pwa.installTitle')}</p>
      <p className="text-xs text-green-600">{t('pwa.installDesc')}</p>
    </div>
    <button
      onClick={install}
      className="text-sm font-bold text-green-700 bg-green-100 px-3 py-1 rounded-lg"
    >
      {t('pwa.install')}
    </button>
  </div>
)}
```

### Step 5 — SW update notification

In `src/main.tsx`, handle the service worker update lifecycle:

```typescript
import { registerSW } from 'virtual:pwa-register'

const updateSW = registerSW({
  onNeedRefresh() {
    // Show update available toast
    toast(t('pwa.updateAvailable'), {
      action: { label: t('pwa.update'), onClick: () => updateSW(true) },
      duration: Infinity,
    })
  },
})
```

### Step 6 — i18n keys

```
pwa.installTitle    → "Installer l'application" / "Install the app"
pwa.installDesc     → "Accédez à Mombongo directement depuis votre écran d'accueil" / "Access Mombongo directly from your home screen"
pwa.install         → "Installer" / "Install"
pwa.updateAvailable → "Une mise à jour est disponible" / "An update is available"
pwa.update          → "Mettre à jour" / "Update"
```

---

## ✅ Definition of Done
- [ ] `vite-plugin-pwa` installed and configured with correct manifest
- [ ] App icons present in `public/icons/` at all required sizes
- [ ] Lighthouse PWA score ≥ 90
- [ ] Install prompt banner shown on supported devices
- [ ] SW update toast shown when new version deployed
- [ ] `OfflineBanner` (from S1-03) still working correctly
- [ ] `npm run build` exits 0

```bash
git commit -m "feat(s7-01): PWA manifest, service worker, install prompt"
git push origin feature/s7-01-pwa-offline
```
