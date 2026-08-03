# S0-02 — Firebase Client, Types & Security Rules

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S0-02 |
| Sprint | Sprint 0 — Infrastructure |
| Branch | `feature/s0-02-firebase-setup` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 3 hours |
| Dependencies | S0-01 |
| Priority | P0 — Blocker for all data stories |

---

## Context
Firebase installed. Test setup done. Now: initialize the Firebase client,
define all TypeScript interfaces for Firestore collections, write Security Rules,
and create seed data for the emulator.

---

## Step 1 — Firebase Client + TypeScript Types

### Lovable Prompt
```
Create src/lib/firebase.ts — full Firebase client.

Initialize Firebase app from env vars:
  VITE_FIREBASE_API_KEY, VITE_FIREBASE_AUTH_DOMAIN, VITE_FIREBASE_PROJECT_ID,
  VITE_FIREBASE_STORAGE_BUCKET, VITE_FIREBASE_MESSAGING_SENDER_ID, VITE_FIREBASE_APP_ID

Export: app, db (Firestore), auth (Auth), storage, functions (region: europe-west1)

After getFirestore(app): call enableIndexedDbPersistence(db).catch(err => {
  if (err.code === 'failed-precondition') console.warn('Multiple tabs open')
  if (err.code === 'unimplemented') console.warn('Browser does not support persistence')
})

Export: isDevMode() → import.meta.env.VITE_DEV_MODE === 'true'

Create src/types/index.ts with ALL of these (use Timestamp from firebase/firestore,
all optional fields use | null not ?):

type UserRole = 'investor' | 'farmer' | 'merchant' | 'agent' | 'admin'
type PaymentMethod = 'mpesa' | 'airtel' | 'orange' | 'equity' | 'stripe'
type PaymentStatus = 'pending' | 'confirmed' | 'failed' | 'refunded'
type ProductStatus = 'draft' | 'open' | 'funded' | 'active' | 'harvesting' | 'completed' | 'cancelled'

interface UserProfile {
  id: string; fullName: string; email: string; phone: string
  role: UserRole; preferredLanguage: 'fr' | 'en' | 'ln'
  avatarUrl: string | null; kycStatus: 'pending' | 'verified' | 'rejected'
  kycVerifiedAt: Timestamp | null; mobileMoneyNumber: string | null
  mobileMoneyProvider: PaymentMethod | null; fcmTokens: string[]
  totalInvestedUsd: number; totalEarnedUsd: number
  referralCode: string; referredBy: string | null; isActive: boolean
  createdAt: Timestamp; updatedAt: Timestamp
}

interface Product {
  id: string; name: string; nameEn: string; nameLn: string
  description: string; descriptionEn: string
  category: 'agriculture' | 'logistique' | 'export' | 'peche' | 'elevage'
  iconEmoji: string; imageUrl: string | null
  supplierId: string; supplierName: string; location: string; region: string
  minInvestmentUsd: number; targetAmountUsd: number; fundedAmountUsd: number
  roiPercentage: number; durationDays: number
  startDate: Timestamp; harvestDate: Timestamp
  investorCount: number; unit: string; stockQuantity: number
  status: ProductStatus; isFeatured: boolean
  createdAt: Timestamp; updatedAt: Timestamp
}

interface Investment {
  id: string; investorId: string; productId: string
  amountUsd: number; amountCdf: number; exchangeRate: number
  roiPercentage: number; expectedReturnUsd: number; actualReturnUsd: number | null
  paymentMethod: PaymentMethod; paymentReference: string | null
  paymentStatus: PaymentStatus
  status: 'pending' | 'active' | 'maturing' | 'completed' | 'cancelled'
  investedAt: Timestamp; maturesAt: Timestamp; completedAt: Timestamp | null
  createdAt: Timestamp
}

interface BourseOpportunity {
  id: string; productName: string; productIcon: string
  farmerId: string; farmerName: string; routeFrom: string; routeTo: string
  quantity: number; unit: string; pricePerUnitCdf: number; totalTransportCdf: number
  commissionPercent: number; minInvestmentCdf: number
  departureDate: Timestamp; estimatedSaleDays: number
  fundedPercent: number; totalInvestors: number
  status: 'open' | 'funded' | 'in_transit' | 'sold' | 'distributed' | 'cancelled'
  createdAt: Timestamp
}

interface BoursePrice {
  id: string; productName: string; priceCdf: number; unit: string
  market: string; changePercent: number; recordedAt: Timestamp
}

interface Course {
  id: string; title: string; titleEn: string; titleLn: string
  description: string; category: 'commerce' | 'agriculture' | 'finance' | 'entrepreneuriat' | 'export'
  level: 'debutant' | 'intermediaire' | 'avance'
  iconEmoji: string; totalLessons: number; totalHours: number
  languages: string[]; enrolledCount: number; rating: number
  isPremium: boolean; isPublished: boolean; instructorName: string
  createdAt: Timestamp
}

interface Transaction {
  id: string; userId: string
  type: 'investment' | 'bourse' | 'withdrawal' | 'profit' | 'refund' | 'fee'
  referenceId: string; amountUsd: number | null; amountCdf: number | null
  paymentMethod: PaymentMethod; externalReference: string | null
  status: PaymentStatus; description: string; metadata: Record<string, unknown>
  createdAt: Timestamp
}

interface Notification {
  id: string; userId: string; type: string
  title: string; titleEn: string; titleLn: string
  message: string; messageEn: string; messageLn: string
  actionUrl: string | null; isRead: boolean; pushSent: boolean
  createdAt: Timestamp
}

interface Farmer {
  id: string; fullName: string; phone: string; location: string; region: string
  totalHectares: number; mainCrop: string; agentId: string | null
  mombongoRating: number; totalFinancedUsd: number; repaymentRate: number
  isVerified: boolean; notes: string; createdAt: Timestamp
}

interface Enrollment {
  id: string; userId: string; courseId: string
  progressPercent: number; lastModule: number
  enrolledAt: Timestamp; completedAt: Timestamp | null; certificateIssued: boolean
}

interface UniversityClub {
  id: string; name: string; fullName: string; city: string; country: string
  memberCount: number; isActive: boolean
}

Export all types from src/types/index.ts.
```

### Regression
```bash
bun run typecheck
# Must exit 0
```

---

## Step 2 — Firestore Security Rules

### Lovable Prompt
```
Create firestore.rules in the project root (replace any existing content):

rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAuthenticated() { return request.auth != null; }
    function isAdmin() { return isAuthenticated() && getUserRole() == 'admin'; }
    function isAgent() { return isAuthenticated() && getUserRole() == 'agent'; }
    function getUserRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    function isOwner(userId) { return isAuthenticated() && request.auth.uid == userId; }

    match /users/{userId} {
      allow read: if isOwner(userId) || isAdmin();
      allow create: if isAuthenticated() && request.auth.uid == userId;
      allow update: if isOwner(userId) || isAdmin();
      allow delete: if false;
    }
    match /products/{productId} {
      allow read: if isAuthenticated() && resource.data.status != 'draft';
      allow create, update, delete: if isAdmin();
    }
    match /investments/{investmentId} {
      allow read: if isOwner(resource.data.investorId) || isAdmin();
      allow create: if isAuthenticated()
        && request.auth.uid == request.resource.data.investorId
        && request.resource.data.amountUsd > 0
        && request.resource.data.paymentStatus == 'pending';
      allow update: if isAdmin();
      allow delete: if false;
    }
    match /farmers/{farmerId} {
      allow read: if isAuthenticated();
      allow create, update: if isAdmin() || isAgent();
      allow delete: if false;
    }
    match /farmer_financing/{financingId} {
      allow read: if isAuthenticated();
      allow create, update: if isAdmin();
      allow delete: if false;
    }
    match /agent_reports/{reportId} {
      allow read: if isAuthenticated();
      allow create: if isAgent() && request.auth.uid == request.resource.data.agentId;
      allow update, delete: if false;
    }
    match /bourse_opportunities/{opportunityId} {
      allow read: if isAuthenticated();
      allow create, update, delete: if isAdmin();
    }
    match /bourse_investments/{bInvestmentId} {
      allow read: if isOwner(resource.data.investorId) || isAdmin();
      allow create: if isAuthenticated()
        && request.auth.uid == request.resource.data.investorId
        && request.resource.data.amountCdf >= 10000;
      allow update: if isAdmin();
      allow delete: if false;
    }
    match /bourse_prices/{priceId} {
      allow read: if isAuthenticated();
      allow create, update: if isAdmin();
      allow delete: if false;
    }
    match /courses/{courseId} {
      allow read: if isAuthenticated() && resource.data.isPublished == true || isAdmin();
      allow create, update, delete: if isAdmin();
      match /modules/{moduleId} {
        allow read: if isAuthenticated();
        allow create, update, delete: if isAdmin();
      }
    }
    match /enrollments/{enrollmentId} {
      allow read: if isOwner(resource.data.userId) || isAdmin();
      allow create: if isAuthenticated() && request.auth.uid == request.resource.data.userId;
      allow update: if isOwner(resource.data.userId);
      allow delete: if false;
    }
    match /transactions/{transactionId} {
      allow read: if isOwner(resource.data.userId) || isAdmin();
      allow create, update, delete: if false;
    }
    match /notifications/{notificationId} {
      allow read: if isOwner(resource.data.userId);
      allow create, delete: if false;
      allow update: if isOwner(resource.data.userId);
    }
    match /university_clubs/{clubId} {
      allow read: if isAuthenticated();
      allow create, update, delete: if isAdmin();
    }
    match /exchange_rates/{rateId} {
      allow read: if isAuthenticated();
      allow create, update, delete: if false;
    }
  }
}

Create firestore.indexes.json:
{
  "indexes": [
    {"collectionGroup":"products","queryScope":"COLLECTION","fields":[{"fieldPath":"status","order":"ASCENDING"},{"fieldPath":"isFeatured","order":"DESCENDING"},{"fieldPath":"createdAt","order":"DESCENDING"}]},
    {"collectionGroup":"products","queryScope":"COLLECTION","fields":[{"fieldPath":"category","order":"ASCENDING"},{"fieldPath":"status","order":"ASCENDING"},{"fieldPath":"createdAt","order":"DESCENDING"}]},
    {"collectionGroup":"investments","queryScope":"COLLECTION","fields":[{"fieldPath":"investorId","order":"ASCENDING"},{"fieldPath":"status","order":"ASCENDING"},{"fieldPath":"investedAt","order":"DESCENDING"}]},
    {"collectionGroup":"bourse_prices","queryScope":"COLLECTION","fields":[{"fieldPath":"productName","order":"ASCENDING"},{"fieldPath":"recordedAt","order":"DESCENDING"}]},
    {"collectionGroup":"notifications","queryScope":"COLLECTION","fields":[{"fieldPath":"userId","order":"ASCENDING"},{"fieldPath":"isRead","order":"ASCENDING"},{"fieldPath":"createdAt","order":"DESCENDING"}]},
    {"collectionGroup":"transactions","queryScope":"COLLECTION","fields":[{"fieldPath":"userId","order":"ASCENDING"},{"fieldPath":"createdAt","order":"DESCENDING"}]}
  ],
  "fieldOverrides":[]
}
```

### Terminal — deploy to dev
```bash
bunx firebase deploy --only firestore:rules,firestore:indexes --project mombongo-dev
```

---

## Step 3 — Seed Script

### Lovable Prompt
```
Create scripts/seed-firestore.js — Node.js CommonJS script.

Detects emulator mode via process.env.FIRESTORE_EMULATOR_HOST.
In emulator mode: connects to localhost:8080.
Uses setDoc for idempotency (safe to re-run).

Seed this data:

PRODUCTS (12 docs, IDs product-001 to product-012):
product-001: {name:'Pastèques',nameEn:'Watermelons',category:'agriculture',iconEmoji:'🍉',
  supplierId:'farmer-001',supplierName:'Jean-Baptiste Mwamba',location:'Songololo',
  region:'Kongo Central',minInvestmentUsd:200,targetAmountUsd:5000,fundedAmountUsd:3000,
  roiPercentage:22,durationDays:45,investorCount:12,status:'open',isFeatured:true,
  unit:'kg',stockQuantity:5000,description:'Pastèques fraîches de qualité premium',
  descriptionEn:'Premium fresh watermelons',nameEn:'Watermelons',nameLn:'Mbuma ya maza'}
product-002: {name:'Tomates',category:'agriculture',iconEmoji:'🍅',
  supplierName:'Marie Lukusa',location:'Kenge',region:'Kwango',
  minInvestmentUsd:150,targetAmountUsd:3000,fundedAmountUsd:900,
  roiPercentage:18,durationDays:30,status:'open',isFeatured:false}
product-003: {name:'Café Robusta',category:'export',iconEmoji:'☕',
  supplierName:'Café Congo SARL',location:'Kivu',region:'Sud-Kivu',
  minInvestmentUsd:500,targetAmountUsd:15000,fundedAmountUsd:0,
  roiPercentage:35,durationDays:180,status:'open',isFeatured:true}
product-004: {name:'Tilapia du Lac',category:'peche',iconEmoji:'🐟',
  supplierName:'Pêche Kisangani',location:'Kisangani',region:'Tshopo',
  minInvestmentUsd:300,targetAmountUsd:8000,fundedAmountUsd:6400,
  roiPercentage:25,durationDays:60,status:'open',isFeatured:false}
product-005 to product-012: similar structure with categories agriculture, elevage, export.
Mix isFeatured and different ROI values (15% to 40%).

FARMERS (4 docs):
farmer-001: {fullName:'Jean-Baptiste Mwamba',phone:'+243812345678',
  location:'Songololo',region:'Kongo Central',totalHectares:5,
  mainCrop:'Pastèques',agentId:'user-agent',mombongoRating:4.8,
  totalFinancedUsd:12000,repaymentRate:0.98,isVerified:true,notes:''}
farmer-002 to farmer-004: different farmers in different regions.

BOURSE_OPPORTUNITIES (6 docs):
opp-001: {productName:'Tomates',productIcon:'🍅',farmerId:'farmer-001',
  farmerName:'Jean-Baptiste Mwamba',routeFrom:'Songololo',routeTo:'Kinshasa',
  quantity:50,unit:'bacs',pricePerUnitCdf:8500,totalTransportCdf:425000,
  commissionPercent:20,minInvestmentCdf:10000,estimatedSaleDays:8,
  fundedPercent:60,totalInvestors:12,status:'open'}
opp-002 to opp-006: different products and routes.

BOURSE_PRICES (8 docs):
{productName:'TOMATES',priceCdf:8500,unit:'FC/bac',market:'Kinshasa',changePercent:2.3}
{productName:'MAÏS',priceCdf:1200,unit:'FC/kg',market:'Kinshasa',changePercent:-0.8}
{productName:'MANIOC',priceCdf:450,unit:'FC/kg',market:'Matadi',changePercent:1.2}
{productName:'PASTÈQUES',priceCdf:3200,unit:'FC/pièce',market:'Kinshasa',changePercent:5.1}
{productName:'CAFÉ',priceCdf:12000,unit:'FC/kg',market:'Bukavu',changePercent:3.7}
{productName:'TILAPIA',priceCdf:4500,unit:'FC/kg',market:'Kisangani',changePercent:-1.2}
{productName:'ANANAS',priceCdf:1800,unit:'FC/pièce',market:'Kinshasa',changePercent:0.5}
{productName:'HARICOTS',priceCdf:2100,unit:'FC/kg',market:'Lubumbashi',changePercent:4.2}

UNIVERSITY_CLUBS (11 docs):
{name:'UNIKIN',fullName:'Université de Kinshasa',city:'Kinshasa',country:'RDC',memberCount:145,isActive:true}
{name:'UNIKIS',fullName:'Université de Kisangani',city:'Kisangani',country:'RDC',memberCount:87,isActive:true}
{name:'UNILU',fullName:'Université de Lubumbashi',city:'Lubumbashi',country:'RDC',memberCount:112,isActive:true}
{name:'ULPGL',fullName:'Université Libre des Pays des Grands Lacs',city:'Goma',country:'RDC',memberCount:67,isActive:true}
{name:'ISC BOMA',fullName:'Institut Supérieur de Commerce de Boma',city:'Boma',country:'RDC',memberCount:54,isActive:true}
{name:'ISAM',fullName:'Institut Supérieur d\'Architecture de Matadi',city:'Matadi',country:'RDC',memberCount:43,isActive:true}
{name:'UNIKAM',fullName:'Université de Kananga',city:'Kananga',country:'RDC',memberCount:38,isActive:true}
{name:'ISP MBUJI',fullName:'Institut Supérieur Pédagogique Mbuji-Mayi',city:'Mbuji-Mayi',country:'RDC',memberCount:72,isActive:true}
{name:'UB BUTEMBO',fullName:'Université de Butembo',city:'Butembo',country:'RDC',memberCount:61,isActive:true}
{name:'UOB',fullName:'Université Officielle de Bukavu',city:'Bukavu',country:'RDC',memberCount:89,isActive:true}
{name:'UVIRA',fullName:'Institut Supérieur Pédagogique d\'Uvira',city:'Uvira',country:'RDC',memberCount:29,isActive:true}

COURSES (6 docs):
{title:'Gestion Financière Coopérative',titleEn:'Cooperative Financial Management',
  category:'finance',level:'debutant',isPremium:false,isPublished:true,
  enrolledCount:342,rating:4.7,iconEmoji:'📊',totalLessons:12,totalHours:8,
  languages:['fr','en'],instructorName:'Prof. Kabila Mutombo'}
{title:'Commerce International et Export',category:'export',level:'intermediaire',
  isPremium:true,isPublished:true,enrolledCount:128,rating:4.5,iconEmoji:'🌍',
  totalLessons:18,totalHours:14,languages:['fr']}
{title:'Agriculture Durable en RDC',category:'agriculture',level:'debutant',
  isPremium:false,isPublished:true,enrolledCount:215,rating:4.8,iconEmoji:'🌱',
  totalLessons:10,totalHours:6,languages:['fr','ln']}
{title:'Entrepreneuriat et Leadership',category:'entrepreneuriat',level:'intermediaire',
  isPremium:false,isPublished:true,enrolledCount:189,rating:4.6,iconEmoji:'🚀'}
{title:'Marketing Digital Afrique',category:'commerce',level:'avance',
  isPremium:true,isPublished:true,enrolledCount:67,rating:4.3,iconEmoji:'📱'}
{title:'Microfinance et Épargne',category:'finance',level:'debutant',
  isPremium:false,isPublished:true,enrolledCount:298,rating:4.9,iconEmoji:'💰'}

USERS (5 test accounts):
user-investor: {role:'investor',fullName:'Alain Mutombo',email:'investor@test.com',
  phone:'+243891234567',preferredLanguage:'fr',totalInvestedUsd:4850,
  totalEarnedUsd:820,isActive:true,kycStatus:'verified',fcmTokens:[],
  referralCode:'ALAIN2026',referredBy:null}
user-farmer: {role:'farmer',fullName:'Jean-Baptiste Mwamba',email:'farmer@test.com',
  phone:'+243812345678',preferredLanguage:'fr',isActive:true,kycStatus:'verified'}
user-agent: {role:'agent',fullName:'Théodore Kyungu',email:'agent@test.com',
  phone:'+243823456789',preferredLanguage:'fr',isActive:true,kycStatus:'verified'}
user-admin: {role:'admin',fullName:'Patrick Admin',email:'admin@test.com',
  phone:'+243834567890',preferredLanguage:'fr',isActive:true,kycStatus:'verified'}
user-merchant: {role:'merchant',fullName:'Joëlle Kabila',email:'merchant@test.com',
  phone:'+243845678901',preferredLanguage:'fr',isActive:true,kycStatus:'verified'}

Log "✅ Seeded X documents to collection_name" for each collection.
```

### Terminal — run seed + export
```bash
# Start emulator first
bunx firebase emulators:start --only auth,firestore --project mombongo-dev &
sleep 15

# Seed
FIRESTORE_EMULATOR_HOST=localhost:8080 node scripts/seed-firestore.js

# Export for CI use
bunx firebase emulators:export ./emulator-seed --project mombongo-dev

# Kill emulator
kill %1
```

---

## ✅ Milestone — S0-02 Complete
- [x] `src/lib/firebase.ts` — all 5 exports, offline persistence enabled
- [x] `src/types/index.ts` — all interfaces exported
- [ ] `firestore.rules` deployed to mombongo-dev — **requires owner action** (see below)
- [x] `emulator-seed/` folder committed to repo (52 documents verified)
- [x] `bun run typecheck` exits 0

## 🏁 PR Checklist
- [ ] `bun run test:ci` exits 0 — deferred: requires integration tests (S0-03+)
- [x] `bun run build` exits 0
- [x] Security rules: `transactions` collection rejects all client writes
- [x] Seed: 12 products, 4 farmers, 6 bourse, 8 prices, 11 clubs, 6 courses, 5 users ✅

## 📋 Notes
- PR: https://github.com/Teddmab/mombongo-web/pull/2
- Seed script renamed to `seed-firestore.cjs` (project has `"type":"module"`)
- Firestore emulator port set to `8090` locally (8080 was taken by another project's Vite server)
- CI workflow uses `wait-on tcp:localhost:8090` to reliably detect emulator readiness
- `firebase-admin` moved to devDependencies (seed-only tool)

## 🔐 Owner Actions Required
1. `bunx firebase login` — authenticate Firebase CLI
2. Create Firebase project `mombongo-dev` in the Firebase console (or confirm it exists)
3. `bunx firebase deploy --only firestore:rules,firestore:indexes --project mombongo-dev`
4. Create `.env.local` from `.env.example` and fill in your Firebase project credentials
5. Add GitHub secrets to the repo for CI build: `STAGING_FIREBASE_API_KEY`, `STAGING_FIREBASE_AUTH_DOMAIN`, `STAGING_FIREBASE_STORAGE_BUCKET`, `STAGING_FIREBASE_MESSAGING_SENDER_ID`, `STAGING_FIREBASE_APP_ID`
