# S2-00 — Data Foundation: Functions Init + Firestore Deploy + Admin Products

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S2-00 |
| Sprint | Sprint 2 — Marketplace |
| Branch | `feature/s2-00-data-foundation` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 3h |
| Dependencies | S0-02 (Firebase setup) |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-functions` | 🔨 Active | Initialize TypeScript project, deploy Firestore rules/indexes, products seed script |
| `mombongo-admin` | 🔨 Active | AdminProducts screen — list, create, edit, toggle active |
| `mombongo-web` | ✅ Done | Firestore rules + indexes already live in this repo |

---

## mombongo-functions

### Step 1 — Initialize the project

```bash
cd mombongo-functions
npm init -y
npm install firebase-admin firebase-functions
npm install -D typescript @types/node ts-node
npx tsc --init
```

`package.json` engines + scripts:
```json
{
  "engines": { "node": "20" },
  "main": "lib/index.js",
  "scripts": {
    "build": "tsc",
    "serve": "npm run build && firebase emulators:start --only functions",
    "deploy": "npm run build && firebase deploy --only functions"
  }
}
```

`tsconfig.json` key settings:
```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "ES2020",
    "outDir": "lib",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src"]
}
```

### Step 2 — Admin SDK init

Create `src/lib/admin.ts`:
```typescript
import * as admin from 'firebase-admin'

if (!admin.apps.length) {
  admin.initializeApp()
}

export const db = admin.firestore()
export const auth = admin.auth()
export { admin }
```

### Step 3 — Deploy Firestore rules + indexes

Copy `firestore.rules` and `firestore.indexes.json` from `mombongo-web/` into `mombongo-functions/` (or reference them from a shared location). Add to `firebase.json` in the functions project:

```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "functions": {
    "source": ".",
    "runtime": "nodejs20"
  }
}
```

Deploy rules:
```bash
firebase deploy --only firestore:rules,firestore:indexes
```

### Step 4 — Products seed script

Create `src/scripts/seedProducts.ts`:
```typescript
import { db } from '../lib/admin'
import { FieldValue } from 'firebase-admin/firestore'

const products = [
  {
    name: 'Pastèques',
    icon: '🍉',
    category: 'agriculture',
    location: 'Songololo',
    farmer: 'Jean-Baptiste Mwamba',
    roi: 22,
    minInvest: 200,
    duration: 45,
    stock: 180,
    unit: 'bacs',
    isFeatured: true,
    isActive: true,
    status: 'active',
    progress: 0,
    invested: 0,
    image: 'https://images.unsplash.com/photo-1563114773-84221bd62daa?auto=format&fit=crop&w=800&q=80',
    description: 'Culture de pastèques sur 5 hectares fertiles à Songololo.',
    createdAt: FieldValue.serverTimestamp(),
  },
  // ... remaining 11 products from mock.ts with same shape
]

async function seed() {
  const batch = db.batch()
  for (const p of products) {
    const ref = db.collection('products').doc()
    batch.set(ref, p)
  }
  await batch.commit()
  console.log(`Seeded ${products.length} products`)
}

seed().catch(console.error)
```

Run once:
```bash
npx ts-node src/scripts/seedProducts.ts
```

### Step 5 — index.ts entry point

Create `src/index.ts` (empty export for now, functions added per sprint):
```typescript
export * from './investments/createInvestment'  // added in S2-04
```

---

## mombongo-admin

### Current State
AdminDashboard exists. No product management screen yet.

### Step 1 — AdminProducts screen

Create `src/pages/AdminProducts.tsx`:

```typescript
interface ProductRow {
  id: string
  name: string
  category: string
  location: string
  roi: number
  minInvest: number
  isActive: boolean
  isFeatured: boolean
  createdAt: string
}
```

The screen has three views:
- **List**: table of all products with columns name/category/roi/minInvest/status, toggle active button, edit button
- **Create/Edit modal**: form with all product fields (name, icon, category, location, farmer, roi, minInvest, duration, stock, unit, description, image URL, isFeatured, isActive)
- **Delete**: soft-delete via `isActive = false` (no hard deletes per security rules)

Use TanStack Query:
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { collection, getDocs, addDoc, updateDoc, doc, orderBy, query } from 'firebase/firestore'
import { db } from '@/lib/firebase'

export function useAdminProducts() {
  return useQuery({
    queryKey: ['admin-products'],
    queryFn: async () => {
      const snap = await getDocs(query(collection(db, 'products'), orderBy('createdAt', 'desc')))
      return snap.docs.map(d => ({ id: d.id, ...d.data() } as ProductRow))
    },
  })
}

export function useCreateProduct() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (data: Omit<ProductRow, 'id' | 'createdAt'>) =>
      addDoc(collection(db, 'products'), { ...data, createdAt: serverTimestamp() }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['admin-products'] }),
  })
}

export function useUpdateProduct() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: ({ id, ...data }: Partial<ProductRow> & { id: string }) =>
      updateDoc(doc(db, 'products', id), data),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['admin-products'] }),
  })
}
```

### Step 2 — Add route to Admin router

In `src/App.tsx` (or wherever admin routes are defined):
```tsx
<Route path="/products" element={<AdminProducts />} />
```

Add nav link in the admin sidebar.

---

## ✅ Definition of Done
- [ ] `firebase deploy --only firestore:rules,firestore:indexes` exits 0
- [ ] Seed script runs and products appear in Firestore console
- [ ] Admin `/products` lists seeded products
- [ ] Admin can create a new product — it appears in Firestore
- [ ] Admin can toggle `isActive` — product disappears from web `/market` (rules block status=draft/inactive)
- [ ] `npm run build` exits 0 in mombongo-functions

## 🏁 PR Checklist
```bash
# mombongo-functions
npm run build
firebase deploy --only firestore:rules,firestore:indexes
git commit -m "feat(s2-00): init functions project + Firestore deploy + products seed"

# mombongo-admin
npm run build && npm run test:unit
git commit -m "feat(s2-00): AdminProducts CRUD screen"
```
