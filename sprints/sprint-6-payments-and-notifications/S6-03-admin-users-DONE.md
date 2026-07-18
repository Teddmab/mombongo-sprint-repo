# S6-03 — Admin — User Management

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S6-03 |
| Sprint | Sprint 6 — Payments & Notifications |
| Branch | `feature/s6-03-admin-users` |
| Merges into | `dev` |
| Estimate | 2h |
| Dependencies | S6-01 |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-admin` | 🔨 Active | AdminUsers screen — user list, KYC status, disable/enable account |
| `mombongo-functions` | 🔨 Active | setUserRole onCall + disableUser onCall (admin-only) |
| `mombongo-web` | ✅ Done | — |

---

## mombongo-functions

### setUserRole onCall

Create `src/admin/setUserRole.ts`:

```typescript
export const setUserRole = functions.https.onCall(async (data, context) => {
  // Verify caller is admin
  const callerSnap = await db.collection('users').doc(context.auth!.uid).get()
  if (callerSnap.data()?.role !== 'admin')
    throw new functions.https.HttpsError('permission-denied', 'Admin only')

  const { userId, role }: { userId: string; role: 'investor' | 'agent' | 'admin' } = data

  await db.collection('users').doc(userId).update({ role })
  await admin.auth().setCustomUserClaims(userId, { role })

  return { success: true }
})
```

### disableUser onCall

Create `src/admin/disableUser.ts`:

```typescript
export const disableUser = functions.https.onCall(async (data, context) => {
  const callerSnap = await db.collection('users').doc(context.auth!.uid).get()
  if (callerSnap.data()?.role !== 'admin')
    throw new functions.https.HttpsError('permission-denied', 'Admin only')

  const { userId, disabled }: { userId: string; disabled: boolean } = data

  await admin.auth().updateUser(userId, { disabled })
  await db.collection('users').doc(userId).update({ disabled })

  return { success: true }
})
```

Export both in `src/index.ts`.

---

## mombongo-admin

### AdminUsers screen

`src/pages/AdminUsers.tsx`:

**Table columns**: avatar + displayName / email / role chip / kycStatus chip / walletUsd / walletCdf / joinedAt / disabled toggle

```typescript
const [search, setSearch] = useState('')
const { data: snapshot, refetch } = useQuery({
  queryKey: ['admin-users'],
  queryFn: () => getDocs(query(collection(db, 'users'), orderBy('createdAt', 'desc'), limit(100))),
})
const users = (snapshot?.docs.map(d => ({ id: d.id, ...d.data() })) ?? [])
  .filter(u => !search || u.displayName?.toLowerCase().includes(search.toLowerCase())
                       || u.email?.toLowerCase().includes(search.toLowerCase()))
```

**Search bar** at top of page.

**Role chip colors**: `investor` (blue) / `agent` (amber) / `admin` (purple).

**KYC status chips**: `verified` (green) / `pending` (amber) / `rejected` (red) / `none` (gray).

**Actions column**:
```tsx
<div className="flex gap-2">
  <button onClick={() => openUserDrawer(user)}>Détails</button>
  <button onClick={() => handleSetRole(user.id, 'agent')}>→ Agent</button>
  <button
    onClick={() => handleDisable(user.id, !user.disabled)}
    className={user.disabled ? 'text-green-600' : 'text-red-600'}
  >
    {user.disabled ? 'Activer' : 'Désactiver'}
  </button>
</div>
```

**User detail drawer** (slide-in panel):
- Shows all profile fields (displayName, email, phone, role, walletUsd, walletCdf)
- KYC status with update dropdown: `none → pending → verified / rejected`
- Investment history: list from `investments` where `investorId == user.id`
- Transaction history: list from `transactions` where `userId == user.id`

```typescript
async function handleSetRole(userId: string, role: string) {
  await httpsCallable(functions, 'setUserRole')({ userId, role })
  refetch()
}

async function handleDisable(userId: string, disabled: boolean) {
  await httpsCallable(functions, 'disableUser')({ userId, disabled })
  refetch()
}

async function handleKycUpdate(userId: string, kycStatus: string) {
  await updateDoc(doc(db, 'users', userId), { kycStatus })
  refetch()
}
```

Route: add `/users` to admin router (already exists, enhance the implementation).

---

## ✅ Definition of Done
- [ ] Admin user list loads from Firestore with search
- [ ] Role change via `setUserRole` function updates Firestore + Firebase Auth custom claims
- [ ] Disable/enable via `disableUser` function
- [ ] KYC status updatable directly from admin
- [ ] User detail drawer shows investment + transaction history
- [ ] `npm run build` exits 0 (admin)

```bash
firebase deploy --only functions:setUserRole,disableUser
git commit -m "feat(s6-03): admin user management — role change, disable, KYC"
git push origin feature/s6-03-admin-users
```
