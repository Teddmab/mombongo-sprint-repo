# S1-04 — Role-Based Home Screens

## Metadata
| Field | Value |
|-------|-------|
| Story ID | S1-04 |
| Sprint | Sprint 1 — Auth & Home |
| Branch | `feature/s1-04-role-home-screens` |
| Merges into | `dev` |
| Owner | Afrotouch OU |
| Estimate | 5 hours |
| Dependencies | S1-02 (role type = `"merchant"` not `"trader"`), S1-03 (useAuth + userProfile), S0-03 (design system) |
| Priority | P1 — Unblocks Sprint 2–4 role-gated features |

## Repo Scope
| Repo | Status | Work |
|------|--------|------|
| `mombongo-web` | 🔨 Active | `FarmerHome`, `MerchantHome`, `AgentHome` — full desktop + mobile variants |
| `mombongo-admin` | ✅ N/A | Admin has a single fixed layout — no role variants |
| `mombongo-mobile` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-functions` | ⏳ Sprint 2 | Repo not yet initialized |
| `mombongo-backoffice` | ⏳ Sprint 2 | Repo not yet initialized |

---

## mombongo-web

### Current State (already implemented — do NOT rewrite)

`src/pages/HomeScreen.tsx` already has:
- ✅ `InvestorHome` — fully built as `DesktopHome` / `MobileHome` with KPI strip, portfolio chart, active investments table, activity feed, quick actions
- ✅ Role dispatch: `if (role === "farmer") return <FarmerHome />` (line 60)
- ✅ `FarmerHome` stub — renders a single green card with hardcoded farmer name, location, 3 stats

**What is missing (this sprint's work):**
- ❌ `FarmerHome` stub is incomplete: hardcoded `"Jean-Baptiste Mwamba"`, no financing tranche, no crops, no quick actions, no desktop variant
- ❌ No `MerchantHome` — `merchant` role falls through to the investor layout (wrong)
- ❌ No `AgentHome` — `agent` role falls through to the investor layout (wrong)
- ❌ `AppContext.tsx` still uses `"trader"` — S1-02 will rename it to `"merchant"`. This story assumes S1-02 is merged first
- ❌ `HomeScreen` dispatch only covers `"farmer"`, not `"merchant"` or `"agent"`

**Do NOT change `InvestorHome` / `DesktopHome` / `MobileHome`. Only add the three missing role components and update the dispatch.**

---

### Architecture — Role Dispatch

**File: `src/pages/HomeScreen.tsx`**

The dispatch block (currently line 60) must be expanded to cover all 4 roles:

```tsx
export default function HomeScreen() {
  const { t } = useTranslation();
  const { role, userName } = useApp();
  const isDesktop = useIsDesktop();

  if (role === "farmer")   return <div data-testid="home-screen" style={{ display: "contents" }}><FarmerHome isDesktop={isDesktop} /></div>;
  if (role === "merchant") return <div data-testid="home-screen" style={{ display: "contents" }}><MerchantHome isDesktop={isDesktop} /></div>;
  if (role === "agent")    return <div data-testid="home-screen" style={{ display: "contents" }}><AgentHome isDesktop={isDesktop} /></div>;

  // Default: investor
  return (
    <div data-testid="home-screen" style={{ display: "contents" }}>
      {isDesktop ? <DesktopHome /> : <MobileHome />}
    </div>
  );
}
```

> **Note:** After S1-02 is merged, `AppContext.Role` will be `"investor" | "farmer" | "merchant" | "agent"`. If S1-02 is not yet merged and `"trader"` still exists in the type, add `"trader"` as an alias: `if (role === "merchant" || role === "trader")` and remove the alias once S1-02 lands.

---

### Step 1 — FarmerHome (replace existing stub)

The current `FarmerHome` stub starts at line 541. **Replace the entire `FarmerHome` function** (lines 541–565) with the full implementation below. Do not touch anything above line 541.

#### Data needed
Import at top of file (alongside existing imports):
```tsx
import { farmers, products } from "@/data/mock";
import { useAuth } from "@/hooks/useAuth";
```

#### FarmerHome — Desktop

```tsx
function FarmerHome({ isDesktop }: { isDesktop: boolean }) {
  const { t } = useTranslation();
  const { userProfile } = useAuth();
  const navigate = useNavigate();

  // In dev/mock mode, use the first farmer as a stand-in for the logged-in farmer
  const farmer = farmers.find(f => f.name === userProfile?.displayName) ?? farmers[0];
  const myProducts = products.filter(p => p.farmer === farmer.name);
  const pct = Math.round((farmer.raised / farmer.needed) * 100);

  if (isDesktop) {
    return (
      <div>
        <PageTitle
          eyebrow={t("home.farmerEyebrow")}
          title={t("home.farmerTitle")}
          description={t("home.farmerDesc")}
          actions={
            <button
              onClick={() => navigate("/financement")}
              className="h-10 px-4 bg-green-700 text-white rounded-xl font-display font-bold text-[13px] flex items-center gap-2 shadow-elevated"
            >
              <Sprout className="w-4 h-4" /> {t("home.farmerFinanceBtn")}
            </button>
          }
        />

        <div className="grid grid-cols-3 gap-4 mb-5">
          {/* Profile + financing tranche card */}
          <div className="col-span-2 bg-gradient-to-br from-green-700 to-green-900 text-white rounded-2xl p-6 shadow-elevated">
            <div className="flex items-start gap-4">
              <div className="relative w-16 h-16 rounded-2xl overflow-hidden flex-shrink-0 bg-white/15">
                <span className="absolute inset-0 flex items-center justify-center text-3xl">{farmer.avatar}</span>
                {farmer.image && (
                  <img src={farmer.image} alt={farmer.name} className="absolute inset-0 w-full h-full object-cover object-top" onError={(e) => { e.currentTarget.style.display = "none"; }} />
                )}
              </div>
              <div className="flex-1">
                <p className="font-display font-black text-[22px] leading-tight">{farmer.name}</p>
                <p className="text-[12px] text-white/60 mt-0.5 flex items-center gap-1">
                  <MapPin className="w-3 h-3" /> {farmer.location}
                </p>
                <div className="flex flex-wrap gap-1.5 mt-2">
                  {farmer.crops.map(c => (
                    <span key={c} className="text-[10px] bg-white/15 rounded-full px-2 py-0.5 font-bold">{c}</span>
                  ))}
                </div>
              </div>
            </div>

            <div className="grid grid-cols-3 gap-4 mt-5 pt-5 border-t border-white/15">
              <div>
                <p className="text-[10px] text-white/50 uppercase tracking-wider font-bold">{t("home.farmerSurface")}</p>
                <p className="font-display font-extrabold text-[20px] tabular mt-1">{farmer.surface} ha</p>
              </div>
              <div>
                <p className="text-[10px] text-white/50 uppercase tracking-wider font-bold">{t("home.farmerExperience")}</p>
                <p className="font-display font-extrabold text-[20px] tabular mt-1">{farmer.experience} ans</p>
              </div>
              <div>
                <p className="text-[10px] text-white/50 uppercase tracking-wider font-bold">{t("home.farmerTrustScore")}</p>
                <p className="font-display font-extrabold text-[20px] tabular mt-1">{farmer.trustScore}/100</p>
              </div>
            </div>

            <div className="mt-5 pt-5 border-t border-white/15">
              <div className="flex items-baseline justify-between mb-2">
                <p className="text-[11px] text-white/60 font-bold uppercase tracking-wider">{t("home.farmerFinancing")}</p>
                <p className="text-[12px] font-bold text-white">{pct}% {t("home.farmerFinancingCollected")}</p>
              </div>
              <div className="h-2.5 bg-white/20 rounded-full overflow-hidden">
                <div className="h-full bg-white rounded-full" style={{ width: `${pct}%` }} />
              </div>
              <div className="flex items-baseline justify-between mt-2">
                <p className="text-[12px] text-white font-extrabold tabular">${farmer.raised.toLocaleString()}</p>
                <p className="text-[11px] text-white/50 tabular">/ ${farmer.needed.toLocaleString()}</p>
              </div>
            </div>
          </div>

          {/* Quick actions sidebar */}
          <div className="bg-white border border-gray-200 rounded-2xl p-5 shadow-card flex flex-col gap-3">
            <h3 className="font-display font-bold text-[14px]">{t("home.shortcuts")}</h3>
            {[
              { icon: ClipboardCheck, label: t("home.farmerActionReport"), to: "/rapport", tone: "green" },
              { icon: Leaf, label: t("home.farmerActionProducts"), to: "/market", tone: "green" },
              { icon: GraduationCap, label: t("home.farmerActionAcademia"), to: "/academia", tone: "blue" },
              { icon: Sprout, label: t("home.farmerActionFinancement"), to: "/financement", tone: "green" },
            ].map(a => (
              <button
                key={a.to}
                onClick={() => navigate(a.to)}
                className="flex items-center gap-3 p-3 rounded-xl border border-gray-100 hover:border-green-700 hover:bg-green-50 transition group"
              >
                <div className={`w-9 h-9 rounded-xl flex items-center justify-center flex-shrink-0 ${
                  a.tone === "blue" ? "bg-blue-50 text-info" : "bg-green-50 text-green-700"
                } group-hover:scale-105 transition`}>
                  <a.icon className="w-4 h-4" strokeWidth={2} />
                </div>
                <span className="font-display font-bold text-[13px] text-gray-900">{a.label}</span>
                <ChevronRight className="w-3.5 h-3.5 text-gray-300 ml-auto group-hover:text-gray-500 transition" />
              </button>
            ))}
          </div>
        </div>

        {/* My products on market */}
        {myProducts.length > 0 && (
          <div className="bg-white border border-gray-200 rounded-2xl shadow-card overflow-hidden">
            <div className="px-6 py-4 flex items-center justify-between border-b border-gray-100">
              <h3 className="font-display font-bold text-[15px]">{t("home.farmerMyProducts")}</h3>
              <button onClick={() => navigate("/market")} className="text-[11px] font-bold text-green-700 hover:underline flex items-center gap-1">
                {t("home.seeAllLink")} <ChevronRight className="w-3.5 h-3.5" />
              </button>
            </div>
            <div className="grid grid-cols-4 gap-0 divide-x divide-gray-100">
              {myProducts.slice(0, 4).map(p => (
                <button key={p.id} onClick={() => navigate(`/market/${p.id}`)} className="flex flex-col items-start p-5 hover:bg-gray-50 transition text-left">
                  <div className="relative w-10 h-10 rounded-xl overflow-hidden bg-green-50 mb-3">
                    <span className="absolute inset-0 flex items-center justify-center text-xl">{p.icon}</span>
                    {p.image && (
                      <img src={p.image} alt={p.name} className="absolute inset-0 w-full h-full object-cover" onError={(e) => { e.currentTarget.style.display = "none"; }} />
                    )}
                  </div>
                  <p className="font-display font-bold text-[14px] text-gray-900">{p.name}</p>
                  <p className="text-[11px] text-gray-500 mt-0.5">{p.location}</p>
                  <p className="text-[12px] text-green-600 font-extrabold mt-2">+{p.roi}% ROI</p>
                </button>
              ))}
            </div>
          </div>
        )}
      </div>
    );
  }

  // ── MOBILE ──
  return (
    <div className="bg-app min-h-full pb-28">
      {/* Profile card */}
      <section className="mx-3 mt-3 bg-gradient-to-br from-green-700 to-green-800 text-white rounded-2xl p-5">
        <div className="flex items-center gap-3">
          <div className="relative w-14 h-14 rounded-2xl overflow-hidden flex-shrink-0 bg-white/15">
            <span className="absolute inset-0 flex items-center justify-center text-2xl">{farmer.avatar}</span>
            {farmer.image && (
              <img src={farmer.image} alt={farmer.name} className="absolute inset-0 w-full h-full object-cover object-top" onError={(e) => { e.currentTarget.style.display = "none"; }} />
            )}
          </div>
          <div>
            <p className="font-display font-black text-[18px] leading-tight">{farmer.name}</p>
            <p className="text-[11px] text-white/60 mt-0.5 flex items-center gap-1">
              <MapPin className="w-3 h-3" /> {farmer.location}
            </p>
          </div>
        </div>
        <div className="grid grid-cols-3 gap-2 text-center mt-4 pt-4 border-t border-white/20">
          {[
            { l: t("home.farmerSurface"),    v: `${farmer.surface} ha` },
            { l: t("home.farmerTrustScore"), v: `${farmer.trustScore}` },
            { l: t("home.farmerExperience"), v: `${farmer.experience} ans` },
          ].map(s => (
            <div key={s.l}>
              <p className="font-display font-extrabold text-[16px]">{s.v}</p>
              <p className="text-[9px] text-white/60 mt-0.5">{s.l}</p>
            </div>
          ))}
        </div>
      </section>

      {/* Financing tranche */}
      <section className="mx-3 mt-3 bg-white border border-gray-200 rounded-2xl p-4">
        <p className="text-[10px] font-bold text-gray-400 uppercase tracking-wider mb-3">{t("home.farmerFinancing")}</p>
        <div className="flex items-baseline justify-between mb-2">
          <span className="font-display font-extrabold text-[18px] text-gray-900 tabular">${farmer.raised.toLocaleString()}</span>
          <span className="text-[11px] text-gray-400 tabular">/ ${farmer.needed.toLocaleString()}</span>
        </div>
        <div className="h-2 bg-gray-100 rounded-full overflow-hidden">
          <div className="h-full bg-green-700" style={{ width: `${pct}%` }} />
        </div>
        <p className="text-[11px] text-green-700 font-bold mt-1.5">{pct}% {t("home.farmerFinancingCollected")}</p>
      </section>

      {/* Quick actions */}
      <p className="mx-3 mt-4 mb-2 text-[10px] uppercase tracking-wider text-gray-400 font-bold">{t("home.shortcuts")}</p>
      <div className="mx-3 grid grid-cols-2 gap-3">
        {[
          { icon: ClipboardCheck, label: t("home.farmerActionReport"),    to: "/rapport",      tone: "green" },
          { icon: Leaf,           label: t("home.farmerActionProducts"),  to: "/market",       tone: "green" },
          { icon: GraduationCap, label: t("home.farmerActionAcademia"),   to: "/academia",     tone: "blue" },
          { icon: Sprout,        label: t("home.farmerActionFinancement"), to: "/financement",  tone: "green" },
        ].map(a => (
          <button
            key={a.to}
            onClick={() => navigate(a.to)}
            className="flex items-center gap-2.5 p-3.5 bg-white border border-gray-200 rounded-2xl active:scale-[0.97] transition"
          >
            <div className={`w-9 h-9 rounded-xl flex items-center justify-center flex-shrink-0 ${
              a.tone === "blue" ? "bg-blue-50 text-info" : "bg-green-50 text-green-700"
            }`}>
              <a.icon className="w-4 h-4" strokeWidth={2} />
            </div>
            <span className="font-display font-bold text-[12px] text-gray-900 leading-tight">{a.label}</span>
          </button>
        ))}
      </div>

      {/* My products */}
      {myProducts.length > 0 && (
        <>
          <p className="mx-3 mt-5 mb-2 text-[10px] uppercase tracking-wider text-gray-400 font-bold">{t("home.farmerMyProducts")}</p>
          <div className="mx-3 space-y-2">
            {myProducts.slice(0, 3).map(p => (
              <button
                key={p.id}
                onClick={() => navigate(`/market/${p.id}`)}
                className="w-full flex items-center gap-3 bg-white border border-gray-200 rounded-2xl p-3.5 text-left active:scale-[0.98] transition"
              >
                <div className="relative w-11 h-11 rounded-xl overflow-hidden flex-shrink-0 bg-green-50">
                  <span className="absolute inset-0 flex items-center justify-center text-xl">{p.icon}</span>
                  {p.image && (
                    <img src={p.image} alt={p.name} className="absolute inset-0 w-full h-full object-cover" onError={(e) => { e.currentTarget.style.display = "none"; }} />
                  )}
                </div>
                <div className="flex-1 min-w-0">
                  <p className="font-display font-bold text-[14px] text-gray-900">{p.name}</p>
                  <p className="text-[11px] text-gray-500">{p.location} · {p.duration}j</p>
                </div>
                <p className="text-[13px] font-extrabold text-green-600 flex-shrink-0">+{p.roi}%</p>
              </button>
            ))}
          </div>
        </>
      )}
    </div>
  );
}
```

---

### Step 2 — MerchantHome (new component)

Add `MerchantHome` **after** `FarmerHome` in the same file.

#### Data needed
`bourseOpportunities` and `bourseTicker` are already exported from `@/data/mock`. Add them to the existing import:
```tsx
import { farmers, products, bourseOpportunities, bourseTicker } from "@/data/mock";
```

#### MerchantHome — full implementation

```tsx
function MerchantHome({ isDesktop }: { isDesktop: boolean }) {
  const { t } = useTranslation();
  const { userName } = useApp();
  const { userProfile } = useAuth();
  const navigate = useNavigate();
  const firstName = userProfile?.displayName?.split(" ")[0] ?? userName;

  // Merchant sees bourse opportunities (transport/stockage/transformation)
  const activeOpportunities = bourseOpportunities.slice(0, 4);

  if (isDesktop) {
    return (
      <div>
        <PageTitle
          eyebrow={t("home.merchantEyebrow")}
          title={`${t("home.merchantTitle")}, ${firstName}`}
          description={t("home.merchantDesc")}
          actions={
            <button
              onClick={() => navigate("/bourse")}
              className="h-10 px-4 bg-amber-600 text-white rounded-xl font-display font-bold text-[13px] flex items-center gap-2 shadow-elevated"
            >
              <CandlestickChart className="w-4 h-4" /> {t("home.merchantBourseBtn")}
            </button>
          }
        />

        {/* Bourse ticker strip */}
        <div className="bg-gray-900 rounded-2xl px-5 py-3 mb-5 flex items-center gap-6 overflow-x-auto no-scrollbar">
          <span className="text-[10px] font-bold text-amber-400 uppercase tracking-widest flex-shrink-0">BOURSE</span>
          {bourseTicker.map(t => (
            <div key={t.symbol} className="flex items-center gap-2 flex-shrink-0">
              <span className="text-[12px] font-bold text-white tabular">{t.symbol}</span>
              <span className="text-[11px] text-gray-400 tabular">{t.price}</span>
              <span className={`text-[11px] font-extrabold tabular ${t.change >= 0 ? "text-green-400" : "text-red-400"}`}>
                {t.change >= 0 ? "▲" : "▼"} {Math.abs(t.change)}%
              </span>
            </div>
          ))}
        </div>

        <div className="grid grid-cols-3 gap-4 mb-5">
          {/* Bourse opportunities */}
          <div className="col-span-2 bg-white border border-gray-200 rounded-2xl shadow-card overflow-hidden">
            <div className="px-6 py-4 flex items-center justify-between border-b border-gray-100">
              <h3 className="font-display font-bold text-[15px]">{t("home.merchantOpportunities")}</h3>
              <button onClick={() => navigate("/bourse")} className="text-[11px] font-bold text-amber-600 hover:underline flex items-center gap-1">
                {t("home.seeAllLink")} <ChevronRight className="w-3.5 h-3.5" />
              </button>
            </div>
            <div className="divide-y divide-gray-100">
              {activeOpportunities.map(opp => {
                const pctFilled = Math.round(((opp.spotsTotal - opp.spotsLeft) / opp.spotsTotal) * 100);
                const typeIcon = opp.type === "transport" ? Truck : opp.type === "stockage" ? ShoppingBag : Leaf;
                const TypeIcon = typeIcon;
                return (
                  <button
                    key={opp.id}
                    onClick={() => navigate(`/bourse/${opp.id}`)}
                    className="w-full flex items-center gap-4 px-6 py-4 hover:bg-gray-50 transition text-left group"
                  >
                    <div className="w-11 h-11 rounded-xl bg-amber-50 flex items-center justify-center flex-shrink-0">
                      <TypeIcon className="w-5 h-5 text-amber-600" strokeWidth={1.8} />
                    </div>
                    <div className="flex-1 min-w-0">
                      <p className="font-display font-bold text-[14px] text-gray-900">{opp.title}</p>
                      <p className="text-[11px] text-gray-500 mt-0.5">{opp.volume} · {opp.duration}</p>
                      <div className="mt-1.5 flex items-center gap-2">
                        <div className="flex-1 h-1.5 bg-gray-100 rounded-full overflow-hidden">
                          <div className="h-full bg-amber-400 rounded-full" style={{ width: `${pctFilled}%` }} />
                        </div>
                        <span className="text-[10px] font-bold text-gray-400 tabular">{opp.spotsLeft} {t("home.merchantSpotsLeft")}</span>
                      </div>
                    </div>
                    <div className="text-right flex-shrink-0">
                      <p className="font-display font-extrabold text-[15px] text-gray-900 tabular">{opp.price}</p>
                      <p className="text-[11px] font-bold text-amber-600 mt-0.5">+{opp.commission}% ROI</p>
                    </div>
                    <ChevronRight className="w-4 h-4 text-gray-300 group-hover:text-gray-500 transition" />
                  </button>
                );
              })}
            </div>
          </div>

          {/* Merchant quick actions */}
          <div className="bg-white border border-gray-200 rounded-2xl p-5 shadow-card flex flex-col gap-3">
            <h3 className="font-display font-bold text-[14px]">{t("home.shortcuts")}</h3>
            {[
              { icon: CandlestickChart, label: t("home.merchantActionBourse"),    to: "/bourse",      tone: "amber" },
              { icon: Truck,            label: t("home.merchantActionLogistique"), to: "/market",      tone: "amber" },
              { icon: ShoppingBag,      label: t("home.merchantActionExport"),     to: "/market",      tone: "blue" },
              { icon: GraduationCap,    label: t("home.merchantActionAcademia"),   to: "/academia",    tone: "blue" },
            ].map(a => (
              <button
                key={`${a.to}-${a.label}`}
                onClick={() => navigate(a.to)}
                className="flex items-center gap-3 p-3 rounded-xl border border-gray-100 hover:border-amber-400 hover:bg-amber-50 transition group"
              >
                <div className={`w-9 h-9 rounded-xl flex items-center justify-center flex-shrink-0 ${
                  a.tone === "blue" ? "bg-blue-50 text-info" : "bg-amber-50 text-amber-600"
                } group-hover:scale-105 transition`}>
                  <a.icon className="w-4 h-4" strokeWidth={2} />
                </div>
                <span className="font-display font-bold text-[13px] text-gray-900">{a.label}</span>
                <ChevronRight className="w-3.5 h-3.5 text-gray-300 ml-auto" />
              </button>
            ))}
          </div>
        </div>
      </div>
    );
  }

  // ── MOBILE ──
  return (
    <div className="bg-app min-h-full pb-28">
      {/* Bourse ticker strip */}
      <div className="mx-3 mt-3 bg-gray-900 rounded-2xl px-4 py-3 flex items-center gap-4 overflow-x-auto no-scrollbar">
        <span className="text-[9px] font-bold text-amber-400 uppercase tracking-widest flex-shrink-0">BOURSE</span>
        {bourseTicker.slice(0, 5).map(tk => (
          <div key={tk.symbol} className="flex items-center gap-1.5 flex-shrink-0">
            <span className="text-[11px] font-bold text-white">{tk.symbol}</span>
            <span className={`text-[10px] font-bold ${tk.change >= 0 ? "text-green-400" : "text-red-400"}`}>
              {tk.change >= 0 ? "▲" : "▼"}{Math.abs(tk.change)}%
            </span>
          </div>
        ))}
      </div>

      {/* Welcome */}
      <div className="mx-3 mt-3">
        <p className="text-[11px] text-gray-400 font-semibold">{t("home.merchantEyebrow")}</p>
        <p className="font-display font-black text-[22px] text-gray-900 leading-tight mt-0.5">
          {`${t("home.merchantTitle")}, ${firstName}`}
        </p>
      </div>

      {/* Opportunities */}
      <p className="mx-3 mt-4 mb-2 text-[10px] uppercase tracking-wider text-gray-400 font-bold">{t("home.merchantOpportunities")}</p>
      <div className="mx-3 space-y-2.5">
        {activeOpportunities.map(opp => {
          const pctFilled = Math.round(((opp.spotsTotal - opp.spotsLeft) / opp.spotsTotal) * 100);
          const typeIcon = opp.type === "transport" ? Truck : opp.type === "stockage" ? ShoppingBag : Leaf;
          const TypeIcon = typeIcon;
          return (
            <button
              key={opp.id}
              onClick={() => navigate(`/bourse/${opp.id}`)}
              className="w-full bg-white border border-gray-200 rounded-2xl p-4 text-left active:scale-[0.98] transition"
            >
              <div className="flex items-center gap-3">
                <div className="w-10 h-10 rounded-xl bg-amber-50 flex items-center justify-center flex-shrink-0">
                  <TypeIcon className="w-5 h-5 text-amber-600" strokeWidth={1.8} />
                </div>
                <div className="flex-1 min-w-0">
                  <p className="font-display font-bold text-[13px] text-gray-900 leading-tight">{opp.title}</p>
                  <p className="text-[10px] text-gray-500 mt-0.5">{opp.volume} · {opp.duration}</p>
                </div>
                <div className="text-right flex-shrink-0">
                  <p className="font-display font-extrabold text-[14px] text-amber-600">+{opp.commission}%</p>
                  <p className="text-[10px] text-gray-400">{opp.spotsLeft} {t("home.merchantSpotsLeft")}</p>
                </div>
              </div>
              <div className="mt-3">
                <div className="h-1.5 bg-gray-100 rounded-full overflow-hidden">
                  <div className="h-full bg-amber-400 rounded-full" style={{ width: `${pctFilled}%` }} />
                </div>
              </div>
            </button>
          );
        })}
      </div>

      {/* Quick actions */}
      <p className="mx-3 mt-5 mb-2 text-[10px] uppercase tracking-wider text-gray-400 font-bold">{t("home.shortcuts")}</p>
      <div className="mx-3 grid grid-cols-2 gap-3">
        {[
          { icon: CandlestickChart, label: t("home.merchantActionBourse"),    to: "/bourse",   tone: "amber" },
          { icon: Truck,            label: t("home.merchantActionLogistique"), to: "/market",   tone: "amber" },
          { icon: ShoppingBag,      label: t("home.merchantActionExport"),     to: "/market",   tone: "blue" },
          { icon: GraduationCap,    label: t("home.merchantActionAcademia"),   to: "/academia", tone: "blue" },
        ].map(a => (
          <button
            key={`${a.to}-${a.label}`}
            onClick={() => navigate(a.to)}
            className="flex items-center gap-2.5 p-3.5 bg-white border border-gray-200 rounded-2xl active:scale-[0.97] transition"
          >
            <div className={`w-9 h-9 rounded-xl flex items-center justify-center flex-shrink-0 ${
              a.tone === "blue" ? "bg-blue-50 text-info" : "bg-amber-50 text-amber-600"
            }`}>
              <a.icon className="w-4 h-4" strokeWidth={2} />
            </div>
            <span className="font-display font-bold text-[12px] text-gray-900 leading-tight">{a.label}</span>
          </button>
        ))}
      </div>
    </div>
  );
}
```

---

### Step 3 — AgentHome (new component)

Add `AgentHome` after `MerchantHome`. The agent is a terrain agent assigned to multiple farmers. They need to submit reports, check in farmers, and see their task list for the day.

#### Data needed
`farmers` is already imported. Add `activity` if not already imported:
```tsx
import { farmers, products, bourseOpportunities, bourseTicker, activity } from "@/data/mock";
```

#### AgentHome — full implementation

```tsx
function AgentHome({ isDesktop }: { isDesktop: boolean }) {
  const { t } = useTranslation();
  const { userName } = useApp();
  const { userProfile } = useAuth();
  const navigate = useNavigate();
  const firstName = userProfile?.displayName?.split(" ")[0] ?? userName;

  // In mock mode: agent is assigned to all farmers (real: filtered by agentId from Firestore)
  const assignedFarmers = farmers;
  const pendingReports = assignedFarmers.filter((_, i) => i < 2); // mock: 2 pending reports
  const completedToday = assignedFarmers.length - pendingReports.length;

  if (isDesktop) {
    return (
      <div>
        <PageTitle
          eyebrow={t("home.agentEyebrow")}
          title={`${t("home.agentTitle")}, ${firstName}`}
          description={t("home.agentDesc")}
          actions={
            <button
              onClick={() => navigate("/rapport")}
              className="h-10 px-4 bg-green-700 text-white rounded-xl font-display font-bold text-[13px] flex items-center gap-2 shadow-elevated"
            >
              <ClipboardCheck className="w-4 h-4" /> {t("home.agentNewReport")}
            </button>
          }
        />

        {/* KPI strip */}
        <div className="grid grid-cols-4 gap-4 mb-5">
          {[
            { icon: Sprout,        label: t("home.agentKpiFarmers"),        value: String(assignedFarmers.length), accent: "green" },
            { icon: ClipboardCheck, label: t("home.agentKpiPendingReports"), value: String(pendingReports.length),  accent: "amber" },
            { icon: TrendingUp,    label: t("home.agentKpiCompletedToday"), value: String(completedToday),          accent: "green" },
            { icon: MapPin,        label: t("home.agentKpiZone"),           value: "Kongo Central",                accent: "blue" },
          ].map(k => (
            <div key={k.label} className={`bg-white border border-gray-200 rounded-2xl p-5 shadow-card`}>
              <div className={`w-10 h-10 rounded-xl flex items-center justify-center mb-3 ${
                k.accent === "amber" ? "bg-amber-50 text-amber-600" :
                k.accent === "blue" ? "bg-blue-50 text-info" : "bg-green-50 text-green-700"
              }`}>
                <k.icon className="w-5 h-5" strokeWidth={2} />
              </div>
              <p className="font-display font-black text-[26px] tabular text-gray-900 leading-none">{k.value}</p>
              <p className="text-[11px] text-gray-500 font-semibold mt-1.5">{k.label}</p>
            </div>
          ))}
        </div>

        <div className="grid grid-cols-3 gap-4">
          {/* Assigned farmers list */}
          <div className="col-span-2 bg-white border border-gray-200 rounded-2xl shadow-card overflow-hidden">
            <div className="px-6 py-4 flex items-center justify-between border-b border-gray-100">
              <h3 className="font-display font-bold text-[15px]">{t("home.agentMyFarmers")}</h3>
              <span className="text-[11px] font-bold text-gray-400">{assignedFarmers.length} {t("home.agentFarmersCount")}</span>
            </div>
            <div className="divide-y divide-gray-100">
              {assignedFarmers.map((f, i) => {
                const hasPending = i < pendingReports.length;
                return (
                  <button
                    key={f.id}
                    onClick={() => navigate(`/financement/${f.id}`)}
                    className="w-full flex items-center gap-4 px-6 py-4 hover:bg-gray-50 transition text-left group"
                  >
                    <div className="relative w-11 h-11 rounded-2xl overflow-hidden flex-shrink-0 bg-green-50">
                      <span className="absolute inset-0 flex items-center justify-center text-xl">{f.avatar}</span>
                      {f.image && (
                        <img src={f.image} alt={f.name} className="absolute inset-0 w-full h-full object-cover object-top" onError={(e) => { e.currentTarget.style.display = "none"; }} />
                      )}
                    </div>
                    <div className="flex-1 min-w-0">
                      <p className="font-display font-bold text-[14px] text-gray-900">{f.name}</p>
                      <p className="text-[11px] text-gray-500 mt-0.5 flex items-center gap-1">
                        <MapPin className="w-3 h-3" /> {f.location}
                      </p>
                    </div>
                    <div className="text-right flex-shrink-0">
                      {hasPending ? (
                        <span className="inline-flex items-center gap-1 text-[10px] font-bold bg-amber-50 text-amber-700 rounded-full px-2.5 py-1">
                          ⏳ {t("home.agentPendingReport")}
                        </span>
                      ) : (
                        <span className="inline-flex items-center gap-1 text-[10px] font-bold bg-green-50 text-green-700 rounded-full px-2.5 py-1">
                          ✅ {t("home.agentReportDone")}
                        </span>
                      )}
                    </div>
                    <ChevronRight className="w-4 h-4 text-gray-300 group-hover:text-gray-500 transition" />
                  </button>
                );
              })}
            </div>
          </div>

          {/* Agent quick actions */}
          <div className="bg-white border border-gray-200 rounded-2xl p-5 shadow-card flex flex-col gap-3">
            <h3 className="font-display font-bold text-[14px]">{t("home.shortcuts")}</h3>
            {[
              { icon: ClipboardCheck, label: t("home.agentActionReport"),    to: "/rapport",      tone: "green" },
              { icon: MapPin,         label: t("home.agentActionCheckIn"),   to: "/rapport",      tone: "green" },
              { icon: Sprout,         label: t("home.agentActionFinancing"), to: "/financement",  tone: "green" },
              { icon: GraduationCap, label: t("home.agentActionAcademia"),   to: "/academia",     tone: "blue" },
            ].map(a => (
              <button
                key={`${a.to}-${a.label}`}
                onClick={() => navigate(a.to)}
                className="flex items-center gap-3 p-3 rounded-xl border border-gray-100 hover:border-green-700 hover:bg-green-50 transition group"
              >
                <div className={`w-9 h-9 rounded-xl flex items-center justify-center flex-shrink-0 ${
                  a.tone === "blue" ? "bg-blue-50 text-info" : "bg-green-50 text-green-700"
                } group-hover:scale-105 transition`}>
                  <a.icon className="w-4 h-4" strokeWidth={2} />
                </div>
                <span className="font-display font-bold text-[13px] text-gray-900">{a.label}</span>
                <ChevronRight className="w-3.5 h-3.5 text-gray-300 ml-auto" />
              </button>
            ))}

            {/* Pending reports alert */}
            {pendingReports.length > 0 && (
              <div className="mt-2 p-3.5 bg-amber-50 border border-amber-200 rounded-xl">
                <p className="text-[12px] font-bold text-amber-900">
                  ⚠️ {pendingReports.length} {t("home.agentPendingAlert")}
                </p>
                <p className="text-[11px] text-amber-700 mt-0.5 leading-snug">
                  {t("home.agentPendingAlertBody")}
                </p>
              </div>
            )}
          </div>
        </div>
      </div>
    );
  }

  // ── MOBILE ──
  return (
    <div className="bg-app min-h-full pb-28">
      {/* Header */}
      <div className="mx-3 mt-3">
        <p className="text-[11px] text-gray-400 font-semibold">{t("home.agentEyebrow")}</p>
        <p className="font-display font-black text-[22px] text-gray-900 leading-tight mt-0.5">
          {`${t("home.agentTitle")}, ${firstName}`}
        </p>
      </div>

      {/* KPI mini strip */}
      <div className="mx-3 mt-3 grid grid-cols-3 gap-2">
        {[
          { label: t("home.agentKpiFarmers"),        value: String(assignedFarmers.length), accent: "green" },
          { label: t("home.agentKpiPendingReports"), value: String(pendingReports.length),  accent: "amber" },
          { label: t("home.agentKpiCompletedToday"), value: String(completedToday),          accent: "green" },
        ].map(k => (
          <div key={k.label} className={`bg-white border rounded-2xl p-3 text-center ${
            k.accent === "amber" ? "border-amber-200" : "border-gray-200"
          }`}>
            <p className={`font-display font-black text-[22px] tabular ${
              k.accent === "amber" ? "text-amber-600" : "text-gray-900"
            }`}>{k.value}</p>
            <p className="text-[9px] text-gray-500 font-semibold mt-0.5 leading-tight">{k.label}</p>
          </div>
        ))}
      </div>

      {/* Pending reports alert */}
      {pendingReports.length > 0 && (
        <div className="mx-3 mt-3 bg-amber-50 border border-amber-200 rounded-2xl p-4">
          <p className="text-[13px] font-bold text-amber-900">⚠️ {pendingReports.length} {t("home.agentPendingAlert")}</p>
          <p className="text-[11px] text-amber-700 mt-0.5">{t("home.agentPendingAlertBody")}</p>
          <button
            onClick={() => navigate("/rapport")}
            className="mt-3 w-full h-9 bg-amber-600 text-white rounded-xl font-display font-bold text-[12px]"
          >
            {t("home.agentNewReport")}
          </button>
        </div>
      )}

      {/* Assigned farmers */}
      <p className="mx-3 mt-4 mb-2 text-[10px] uppercase tracking-wider text-gray-400 font-bold">{t("home.agentMyFarmers")}</p>
      <div className="mx-3 space-y-2.5">
        {assignedFarmers.map((f, i) => {
          const hasPending = i < pendingReports.length;
          return (
            <button
              key={f.id}
              onClick={() => navigate(`/financement/${f.id}`)}
              className="w-full flex items-center gap-3 bg-white border border-gray-200 rounded-2xl p-3.5 text-left active:scale-[0.98] transition"
            >
              <div className="relative w-11 h-11 rounded-2xl overflow-hidden flex-shrink-0 bg-green-50">
                <span className="absolute inset-0 flex items-center justify-center text-xl">{f.avatar}</span>
                {f.image && (
                  <img src={f.image} alt={f.name} className="absolute inset-0 w-full h-full object-cover object-top" onError={(e) => { e.currentTarget.style.display = "none"; }} />
                )}
              </div>
              <div className="flex-1 min-w-0">
                <p className="font-display font-bold text-[14px] text-gray-900">{f.name}</p>
                <p className="text-[10px] text-gray-500 flex items-center gap-1 mt-0.5">
                  <MapPin className="w-3 h-3" /> {f.location}
                </p>
              </div>
              {hasPending ? (
                <span className="text-[10px] font-bold bg-amber-50 text-amber-700 rounded-full px-2 py-0.5 flex-shrink-0">⏳ {t("home.agentPendingReport")}</span>
              ) : (
                <span className="text-[10px] font-bold bg-green-50 text-green-700 rounded-full px-2 py-0.5 flex-shrink-0">✅ {t("home.agentReportDone")}</span>
              )}
            </button>
          );
        })}
      </div>

      {/* Quick actions */}
      <p className="mx-3 mt-5 mb-2 text-[10px] uppercase tracking-wider text-gray-400 font-bold">{t("home.shortcuts")}</p>
      <div className="mx-3 grid grid-cols-2 gap-3">
        {[
          { icon: ClipboardCheck, label: t("home.agentActionReport"),    to: "/rapport",     tone: "green" },
          { icon: MapPin,         label: t("home.agentActionCheckIn"),   to: "/rapport",     tone: "green" },
          { icon: Sprout,         label: t("home.agentActionFinancing"), to: "/financement", tone: "green" },
          { icon: GraduationCap,  label: t("home.agentActionAcademia"),  to: "/academia",    tone: "blue" },
        ].map(a => (
          <button
            key={`${a.to}-${a.label}`}
            onClick={() => navigate(a.to)}
            className="flex items-center gap-2.5 p-3.5 bg-white border border-gray-200 rounded-2xl active:scale-[0.97] transition"
          >
            <div className={`w-9 h-9 rounded-xl flex items-center justify-center flex-shrink-0 ${
              a.tone === "blue" ? "bg-blue-50 text-info" : "bg-green-50 text-green-700"
            }`}>
              <a.icon className="w-4 h-4" strokeWidth={2} />
            </div>
            <span className="font-display font-bold text-[12px] text-gray-900 leading-tight">{a.label}</span>
          </button>
        ))}
      </div>
    </div>
  );
}
```

---

### Step 4 — i18n keys

Add the following keys to `src/locales/fr.json` (and matching empty strings to `en.json` / `ln.json`):

```json
{
  "home": {
    "farmerEyebrow": "Mon Exploitation",
    "farmerTitle": "Tableau de bord agriculteur",
    "farmerDesc": "Suivez vos cultures, vos financements et vos produits sur le marché.",
    "farmerFinanceBtn": "Demander un financement",
    "farmerSurface": "Surface",
    "farmerExperience": "Expérience",
    "farmerTrustScore": "Score confiance",
    "farmerFinancing": "Financement en cours",
    "farmerFinancingCollected": "collecté",
    "farmerMyProducts": "Mes produits au marché",
    "farmerActionReport": "Rapport terrain",
    "farmerActionProducts": "Mes produits",
    "farmerActionAcademia": "Formation",
    "farmerActionFinancement": "Financement",

    "merchantEyebrow": "Commerce & Bourse",
    "merchantTitle": "Bonjour",
    "merchantDesc": "Explorez les opportunités de transport, stockage et transformation.",
    "merchantBourseBtn": "Voir la Bourse",
    "merchantOpportunities": "Opportunités actives",
    "merchantSpotsLeft": "places restantes",
    "merchantActionBourse": "La Bourse",
    "merchantActionLogistique": "Logistique",
    "merchantActionExport": "Export",
    "merchantActionAcademia": "Formation",

    "agentEyebrow": "Agent de terrain",
    "agentTitle": "Bonjour",
    "agentDesc": "Gérez vos visites, soumettez vos rapports et suivez vos agriculteurs.",
    "agentNewReport": "Nouveau rapport",
    "agentKpiFarmers": "Agriculteurs",
    "agentKpiPendingReports": "Rapports en attente",
    "agentKpiCompletedToday": "Complétés aujourd'hui",
    "agentKpiZone": "Zone",
    "agentMyFarmers": "Mes agriculteurs",
    "agentFarmersCount": "assignés",
    "agentPendingReport": "Rapport en attente",
    "agentReportDone": "Rapport soumis",
    "agentPendingAlert": "rapports en attente",
    "agentPendingAlertBody": "Soumettez vos rapports de visite avant 18h00.",
    "agentActionReport": "Soumettre rapport",
    "agentActionCheckIn": "Check-in GPS",
    "agentActionFinancing": "Mes financements",
    "agentActionAcademia": "Formation"
  }
}
```

> **Lingala (`ln.json`) note:** Mark all new keys with `"TODO: translate"` for now. They will be filled in Sprint 7 polish pass.

---

### Step 5 — Imports audit

After adding the three new home functions, verify all icons used are already imported at the top of `HomeScreen.tsx`. The new functions use:

| Icon | Already imported? |
|------|------------------|
| `Sprout` | ✅ Yes |
| `Leaf` | ✅ Yes |
| `Truck` | ✅ Yes |
| `ShoppingBag` | ✅ Yes |
| `CandlestickChart` | ✅ Yes |
| `GraduationCap` | ✅ Yes |
| `ClipboardCheck` | ✅ Yes |
| `MapPin` | ✅ Yes |
| `ChevronRight` | ✅ Yes |
| `TrendingUp` | ✅ Yes |

All icons are already in the file. No new lucide imports required.

---

### Unit Tests

File: `src/pages/__tests__/HomeScreen.roleVariants.test.tsx`

```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import { MemoryRouter } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import HomeScreen from '@/pages/HomeScreen'

// Mock auth
vi.mock('@/hooks/useAuth', () => ({
  useAuth: () => ({
    userProfile: { displayName: 'Alain Mutombo', totalInvestedUsd: 0, totalEarnedUsd: 0 },
    isAuthenticated: true,
    isLoading: false,
  })
}))

// Mock products hook
vi.mock('@/hooks/useProducts', () => ({
  useProducts: () => ({ data: [], isLoading: false }),
  useFeaturedProducts: () => ({ data: [], isLoading: false }),
}))

// Shared render helper — role is passed via AppContext mock
function renderAs(role: string) {
  vi.doMock('@/context/AppContext', () => ({
    useApp: () => ({ role, userName: 'Alain', lang: 'fr', setLang: vi.fn(), setRole: vi.fn() }),
    AppProvider: ({ children }: any) => children,
  }))
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  return render(
    <QueryClientProvider client={qc}>
      <MemoryRouter><HomeScreen /></MemoryRouter>
    </QueryClientProvider>
  )
}

describe('HomeScreen — role dispatch', () => {
  it('renders home-screen testid for investor', async () => {
    vi.doMock('@/context/AppContext', () => ({
      useApp: () => ({ role: 'investor', userName: 'Alain', lang: 'fr', setLang: vi.fn(), setRole: vi.fn() }),
      AppProvider: ({ children }: any) => children,
    }))
    const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } })
    render(<QueryClientProvider client={qc}><MemoryRouter><HomeScreen /></MemoryRouter></QueryClientProvider>)
    expect(screen.getByTestId('home-screen')).toBeInTheDocument()
  })

  it('renders home-screen testid for farmer', () => {
    renderAs('farmer')
    expect(screen.getByTestId('home-screen')).toBeInTheDocument()
  })

  it('renders home-screen testid for merchant', () => {
    renderAs('merchant')
    expect(screen.getByTestId('home-screen')).toBeInTheDocument()
  })

  it('renders home-screen testid for agent', () => {
    renderAs('agent')
    expect(screen.getByTestId('home-screen')).toBeInTheDocument()
  })
})

describe('FarmerHome content', () => {
  it('renders farmer financing section', () => {
    renderAs('farmer')
    // Farmer name from mock
    expect(screen.getByText(/Jean-Baptiste Mwamba/i)).toBeInTheDocument()
  })
})

describe('MerchantHome content', () => {
  it('renders BOURSE ticker label', () => {
    renderAs('merchant')
    expect(screen.getAllByText(/BOURSE/i).length).toBeGreaterThan(0)
  })
})

describe('AgentHome content', () => {
  it('renders agent farmers section', () => {
    renderAs('agent')
    // Should show at least one farmer name from mock
    expect(screen.getByText(/Jean-Baptiste Mwamba/i)).toBeInTheDocument()
  })
})
```

### Regression
```bash
bun run test:unit -- src/pages/__tests__/HomeScreen.roleVariants.test.tsx
# Expected: 7 tests pass
bun run build
# Expected: exits 0
```

📝 Manual checklist:
- [ ] In DevTools: set `localStorage.setItem("mb_role", "farmer")` → reload → FarmerHome appears (name, surface, financing bar, crops, products)
- [ ] `localStorage.setItem("mb_role", "merchant")` → reload → MerchantHome appears (bourse ticker, opportunities list)
- [ ] `localStorage.setItem("mb_role", "agent")` → reload → AgentHome appears (farmers list, pending reports alert)
- [ ] `localStorage.setItem("mb_role", "investor")` → reload → InvestorHome appears (KPI strip, chart, investments)
- [ ] Test both desktop (≥1024px) and mobile (375px) for each role
- [ ] DEV_MODE login panel (from S1-02) — switching role from farmer → merchant updates the home screen without full reload

---

## ✅ Milestone — S1-04 Complete
- [ ] **[web]** Role dispatch covers all 4 roles: `investor`, `farmer`, `merchant`, `agent`
- [ ] **[web]** `FarmerHome` shows real farmer profile, financing tranche, crops, product list, quick actions — desktop + mobile
- [ ] **[web]** `MerchantHome` shows bourse ticker, active opportunities with progress bar, quick actions — desktop + mobile
- [ ] **[web]** `AgentHome` shows KPI strip, assigned farmers with report status badges, pending alert, quick actions — desktop + mobile
- [ ] **[web]** 7 unit tests pass
- [ ] **[web]** `bun run build` exits 0
- [ ] **[web]** All new i18n keys added to `fr.json`, `en.json`, `ln.json`
- [ ] **[web]** No TypeScript errors (`bun run typecheck` exits 0)

## 🏁 PR Checklist
- [ ] S1-02 is merged (merchant role type aligned) before this PR is opened
- [ ] `bun run test:ci` exits 0 (web)
- [ ] No hardcoded names or values outside `@/data/mock` — all strings go through i18n `t()`
- [ ] `data-testid="home-screen"` present on root div for all 4 role paths
- [ ] Afrotouch OU review required

```bash
# mombongo-web
git checkout -b feature/s1-04-role-home-screens
git add src/pages/HomeScreen.tsx src/locales/ src/pages/__tests__/HomeScreen.roleVariants.test.tsx
git commit -m "feat(s1-04): role-based home screens — FarmerHome, MerchantHome, AgentHome"
git push origin feature/s1-04-role-home-screens
# PR → dev → merge after S1-02 and S1-03 are merged
```

---

## Future Sprint Notes

> These items are **out of scope for S1-04** and tracked for later sprints:

- **S4-03** will replace mock `assignedFarmers` in `AgentHome` with a real Firestore query filtered by `agentId`
- **S3-01** will replace mock `bourseOpportunities` in `MerchantHome` with a real-time Firestore `onSnapshot` listener
- **S4-01** will replace mock `farmer` profile in `FarmerHome` with the logged-in user's Firestore `farmer_financing` document
- **S7-04** polish pass will fill in all `ln.json` `"TODO: translate"` entries for Lingala
