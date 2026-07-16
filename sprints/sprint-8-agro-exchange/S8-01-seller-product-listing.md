# S8-01 — Agro Exchange — Seller Product Listing

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S8-01 |
| Sprint | Sprint 8 — Agro Exchange |
| Branch | `feature/s8-01-seller-product-listing` |
| Merges into | `dev` |
| Estimate | 4h |
| Dependencies | S8-00 (collections + read CFs exist) |

## Context

Farmers, cooperatives, and exporters need to post what they have for sale. Currently `MettreEnVenteModal` in `ActionForms.tsx` is a stub that calls `setTimeout`. This sprint replaces that stub with a real form that calls a `createProductListing` CF, adds photo upload via a signed-URL CF, and surfaces active listings on the AgricultorBourse screen.

---

## mombongo-functions

### createProductListing onCall

Create `src/bourse/createProductListing.ts`:

```typescript
export const createProductListing = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const {
    commodity, quantityKg, quality, province, territory,
    pricePerKgCdf, availableFrom, availableUntil, description,
  } = data as {
    commodity: string; quantityKg: number; quality: 'A' | 'B' | 'C'
    province: string; territory: string; pricePerKgCdf: number
    availableFrom: string; availableUntil: string; description?: string
  }

  if (quantityKg <= 0) throw new functions.https.HttpsError('invalid-argument', 'Quantité invalide')
  if (pricePerKgCdf <= 0) throw new functions.https.HttpsError('invalid-argument', 'Prix invalide')

  const userSnap = await db.collection('users').doc(uid).get()
  const sellerName = userSnap.data()?.displayName ?? 'Vendeur'
  const sellerRole = userSnap.data()?.role ?? 'farmer'

  const ref = db.collection('product_listings').doc()
  const now = admin.firestore.FieldValue.serverTimestamp()

  await ref.set({
    sellerId: uid,
    sellerName,
    sellerRole,
    commodity,
    quantityKg,
    quality,
    province,
    territory,
    pricePerKgCdf,
    availableFrom: new Date(availableFrom),
    availableUntil: new Date(availableUntil),
    description: description ?? '',
    photoUrls: [],       // uploaded separately via getListingPhotoUploadUrl
    status: 'active',
    createdAt: now,
    updatedAt: now,
  })

  return { listingId: ref.id }
})
```

### getListingPhotoUploadUrl onCall

Create `src/bourse/getListingPhotoUploadUrl.ts`:

```typescript
import { getStorage } from 'firebase-admin/storage'

export const getListingPhotoUploadUrl = functions.region('europe-west1').https.onCall(async (data, context) => {
  const uid = context.auth?.uid
  if (!uid) throw new functions.https.HttpsError('unauthenticated', 'Login required')

  const { listingId, fileName, contentType } = data as {
    listingId: string; fileName: string; contentType: string
  }

  const bucket = getStorage().bucket()
  const filePath = `listings/${uid}/${listingId}/${Date.now()}-${fileName}`
  const file = bucket.file(filePath)

  const [uploadUrl] = await file.getSignedUrl({
    version: 'v4',
    action: 'write',
    expires: Date.now() + 15 * 60 * 1000,  // 15 min
    contentType,
  })

  // Return a signed read URL valid for 7 days (overwritten by a durable URL after upload)
  const [readUrl] = await file.getSignedUrl({
    version: 'v4',
    action: 'read',
    expires: Date.now() + 7 * 24 * 60 * 60 * 1000,
  })

  // Add readUrl to listing.photoUrls
  await db.collection('product_listings').doc(listingId).update({
    photoUrls: admin.firestore.FieldValue.arrayUnion(readUrl),
    updatedAt: admin.firestore.FieldValue.serverTimestamp(),
  })

  return { uploadUrl, readUrl, filePath }
})
```

Export both in `src/index.ts`.

---

## mombongo-web

### Step 1 — useProductListings hook

Create `src/hooks/useProductListings.ts`:

```typescript
import { useQuery } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions, isDevMode } from '@/lib/firebase'
import { myListings as MOCK_LISTINGS } from '@/data/mock'

export interface ProductListing {
  id: string
  sellerId: string
  sellerName: string
  sellerRole: string
  commodity: string
  quantityKg: number
  quality: 'A' | 'B' | 'C'
  province: string
  territory: string
  pricePerKgCdf: number
  availableFrom: { seconds: number }
  availableUntil: { seconds: number }
  description?: string
  photoUrls: string[]
  status: 'active' | 'matched' | 'sold' | 'expired' | 'cancelled'
  createdAt: { seconds: number }
}

export function useProductListings(filters: { commodity?: string; province?: string } = {}) {
  return useQuery({
    queryKey: ['product-listings', filters],
    queryFn: async () => {
      if (isDevMode()) return MOCK_LISTINGS as unknown as ProductListing[]
      const fn = httpsCallable<any, { listings: ProductListing[] }>(functions, 'getProductListings')
      const result = await fn(filters)
      return result.data.listings
    },
    staleTime: 60_000,
  })
}

export function useMyListings() {
  // Same call but filtered by current user's sellerId — CF checks context.auth.uid
  return useQuery({
    queryKey: ['my-listings'],
    queryFn: async () => {
      if (isDevMode()) return MOCK_LISTINGS as unknown as ProductListing[]
      const fn = httpsCallable<any, { listings: ProductListing[] }>(functions, 'getMyProductListings')
      const result = await fn({})
      return result.data.listings
    },
    staleTime: 30_000,
  })
}
```

> **Additional CF needed**: `getMyProductListings` — same as `getProductListings` but filters `sellerId == context.auth.uid`. Add to `src/bourse/getProductListings.ts`.

### Step 2 — Upgrade MettreEnVenteModal

Replace the stub in `src/components/forms/ActionForms.tsx`:

```typescript
import { useQueryClient } from '@tanstack/react-query'
import { httpsCallable } from 'firebase/functions'
import { functions, isDevMode } from '@/lib/firebase'

const createListingFn = httpsCallable<any, { listingId: string }>(functions, 'createProductListing')

export function MettreEnVenteModal({ open, onClose }: { open: boolean; onClose: () => void }) {
  const qc = useQueryClient()
  const [commodity, setCommodity] = useState('Maïs')
  const [quantityKg, setQuantityKg] = useState('')
  const [quality, setQuality] = useState<'A' | 'B' | 'C'>('B')
  const [province, setProvince] = useState('Kinshasa')
  const [territory, setTerritory] = useState('')
  const [pricePerKgCdf, setPricePerKgCdf] = useState('')
  const [availableFrom, setAvailableFrom] = useState('')
  const [availableUntil, setAvailableUntil] = useState('')
  const [description, setDescription] = useState('')
  const [loading, setLoading] = useState(false)
  const [success, setSuccess] = useState(false)

  const DRC_PROVINCES = [
    'Kinshasa', 'Kongo-Central', 'Kwango', 'Kwilu', 'Mai-Ndombe',
    'Kasaï', 'Kasaï-Central', 'Kasaï-Oriental', 'Lomami', 'Sankuru',
    'Maniema', 'Sud-Kivu', 'Nord-Kivu', 'Ituri',
    'Haut-Uele', 'Tshopo', 'Bas-Uele', 'Nord-Ubangi', 'Mongala',
    'Sud-Ubangi', 'Équateur', 'Tshuapa',
    'Tanganyika', 'Haut-Lomami', 'Lualaba', 'Haut-Katanga',
  ]

  const COMMODITIES = ['Maïs', 'Manioc', 'Riz', 'Haricot', 'Cacao', 'Café', 'Palmier', 'Arachide', 'Banane', 'Tomate', 'Pastèque', 'Aubergine', 'Autre']

  const submit = async () => {
    if (!quantityKg || !pricePerKgCdf || !territory || !availableFrom || !availableUntil) {
      toast.error('Remplissez tous les champs obligatoires')
      return
    }
    setLoading(true)
    try {
      if (isDevMode()) {
        await new Promise(r => setTimeout(r, 800))
      } else {
        await createListingFn({
          commodity, quantityKg: Number(quantityKg), quality, province, territory,
          pricePerKgCdf: Number(pricePerKgCdf), availableFrom, availableUntil,
          description,
        })
      }
      await qc.invalidateQueries({ queryKey: ['my-listings'] })
      await qc.invalidateQueries({ queryKey: ['product-listings'] })
      setSuccess(true)
    } catch (e: any) {
      toast.error(e?.message ?? 'Erreur — réessayez')
    } finally {
      setLoading(false)
    }
  }

  const reset = () => {
    setSuccess(false)
    setCommodity('Maïs'); setQuantityKg(''); setQuality('B')
    setProvince('Kinshasa'); setTerritory(''); setPricePerKgCdf('')
    setAvailableFrom(''); setAvailableUntil(''); setDescription('')
  }

  if (success) {
    return (
      <ModalWrap open={open} onClose={() => { reset(); onClose() }} title="Mise en vente">
        <div className="py-8 text-center space-y-3">
          <div className="w-14 h-14 rounded-full bg-green-100 text-green-600 flex items-center justify-center mx-auto">
            <CheckCircle2 className="w-7 h-7" />
          </div>
          <p className="font-display font-bold text-[16px] text-gray-900">Offre publiée !</p>
          <p className="text-[13px] text-gray-500">Votre offre de {quantityKg} kg de {commodity} est visible sur la Bourse.</p>
          <button onClick={() => { reset(); onClose() }} className="mt-4 h-11 px-6 bg-green-700 text-white rounded-xl font-bold text-[13px]">
            Fermer
          </button>
        </div>
      </ModalWrap>
    )
  }

  return (
    <ModalWrap open={open} onClose={onClose} title="Mettre en vente sur la Bourse">
      <div className="space-y-4">
        <Field label="Produit *">
          <select value={commodity} onChange={e => setCommodity(e.target.value)} className={selectCls}>
            {COMMODITIES.map(c => <option key={c}>{c}</option>)}
          </select>
        </Field>
        <div className="grid grid-cols-2 gap-3">
          <Field label="Quantité (kg) *">
            <input value={quantityKg} onChange={e => setQuantityKg(e.target.value)} type="number" min="1" placeholder="20000" className={inputCls} />
          </Field>
          <Field label="Qualité *">
            <select value={quality} onChange={e => setQuality(e.target.value as any)} className={selectCls}>
              <option value="A">A — Premium</option>
              <option value="B">B — Standard</option>
              <option value="C">C — Brut</option>
            </select>
          </Field>
        </div>
        <div className="grid grid-cols-2 gap-3">
          <Field label="Province *">
            <select value={province} onChange={e => setProvince(e.target.value)} className={selectCls}>
              {DRC_PROVINCES.map(p => <option key={p}>{p}</option>)}
            </select>
          </Field>
          <Field label="Territoire *">
            <input value={territory} onChange={e => setTerritory(e.target.value)} placeholder="ex: Kikwit" className={inputCls} />
          </Field>
        </div>
        <Field label="Prix demandé / kg (FC) *">
          <input value={pricePerKgCdf} onChange={e => setPricePerKgCdf(e.target.value)} type="number" min="1" placeholder="400" className={inputCls} />
        </Field>
        <div className="grid grid-cols-2 gap-3">
          <Field label="Disponible dès *">
            <input value={availableFrom} onChange={e => setAvailableFrom(e.target.value)} type="date" className={inputCls} />
          </Field>
          <Field label="Jusqu'au *">
            <input value={availableUntil} onChange={e => setAvailableUntil(e.target.value)} type="date" className={inputCls} />
          </Field>
        </div>
        <Field label="Description (facultatif)">
          <textarea value={description} onChange={e => setDescription(e.target.value)} rows={2} placeholder="Variété, conditions de stockage, mode de livraison…" className={inputCls} />
        </Field>
        <SubmitBtn label="Publier mon offre" icon={Tag} color="green" onClick={submit} loading={loading} />
      </div>
    </ModalWrap>
  )
}
```

### Step 3 — Update AgricultorBourse "Mes produits publiés"

In `src/pages/bourse/AgricultorBourse.tsx`, replace mock `myListings`:

```typescript
import { useMyListings } from '@/hooks/useProductListings'

// Inside component:
const { data: myListings = [], isLoading } = useMyListings()

// Display:
// commodity, quantityKg, quality, pricePerKgCdf, status, province/territory
```

---

## ✅ Definition of Done
- [ ] `createProductListing` CF creates `product_listings` doc in Firestore
- [ ] `getMyProductListings` CF returns current user's listings
- [ ] `MettreEnVenteModal` submits to CF in live mode, mock `setTimeout` in dev mode
- [ ] Success screen appears after submit
- [ ] AgricultorBourse "Mes produits publiés" shows live listings from `useMyListings`
- [ ] `npm run build` exits 0 (web + functions)
- [ ] `npx vitest run` passes

```bash
firebase deploy --only functions:createProductListing,functions:getMyProductListings,functions:getListingPhotoUploadUrl
git commit -m "feat(s8-01): seller product listing — createProductListing CF + upgraded modal"
```
