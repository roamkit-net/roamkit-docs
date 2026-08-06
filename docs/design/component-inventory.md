# Cap2b — Component Inventory (roamkit-web)

| Field | Value |
|-------|-------|
| Status | Draft (feeds Cap2) — **Cap2 CLOSED**; see [status.md](./status.md) and `roamkit-web` Cap2 Review |
| Date | 2026-08 |
| Related | [ADR 016](../adr/016-web-design-tokens.md) |
| Scope | Doc only — no UI code |

## Purpose

After Cap1 (brand/theme tokens), catalog existing style variants so Cap2 Shared UI Primitives migrates **duplicates with decisions**, not discovery.

Each row has a **Status**:

| Status | Meaning |
|--------|---------|
| **Reuse** | Keep as-is or keep as thin domain wrapper over a primitive |
| **Merge** | Collapse into one canonical `ui/*` primitive in Cap2 |
| **Replace** | Needs a different primitive (e.g. Tabs), not a 1:1 Button |
| **Delete** | Unused / superseded (none in this pass) |

**Cap2 stop rule (same as Cap1):** introduce primitives + migrate Merge rows **without** intentional visual change. Visual redesign is Cap3+.

## Roadmap order (locked)

```text
Cap1 Brand Tokens     ✅
Cap2b Inventory       ✅ (this doc — historical)
Cap2 Shared UI Primitives  ✅ CLOSED (API Freeze)
Cap2 Review           ✅ quality gate
Cap3 AppShell Variant A
…
```

## Near-duplicates (Merge first in Cap2)

1. **Primary CTA** — `rounded-lg bg-sky-700` (catalog/eSIM) ↔ `rounded-xl bg-sky-700` (deposit)
2. **Secondary outline** — slate cancel ↔ sky `DepositCta` / dismiss
3. **Auth submit** — `AuthForm` cyan ↔ duplicate on `reset-password`
4. **Auth error** — `AuthForm` red box ↔ set/reset-password copies
5. **Page error amber panel** — identical `rounded-2xl … amber-50 p-6` (~7 files)
6. **Empty state card** — identical `rounded-2xl … p-8 text-center` (4 files)
7. **Pulse skeleton** — private `Pulse` in `ListSkeleton` and `DepositSkeleton`
8. **Link** — `text-sky-700` (~15+) ↔ `DepositCta link` ↔ auth `text-cyan-700`
9. **Modal close** — identical ghost Close in Coverages + Compatibility
10. **List-row card** — `LocationCard` ↔ eSIM list row

## Inventory

| Component | Variant | Count | Pattern | Example | Status | Canonical |
|-----------|---------|------:|---------|---------|--------|-----------|
| Button | Primary (app sky) | ~17 / ~10 files | `rounded-lg bg-sky-700 … hover:bg-sky-800` | `PackageRow.tsx`, `me/esims/[id]/setup` | Merge | `ui/Button` primary |
| Button | Primary (deposit xl) | ~6 / 4 files | `rounded-xl bg-sky-700 …` | `WalletDepositPanel.tsx`, `CexDepositForm.tsx` | Merge | `ui/Button` primary |
| Button | Landing CTA | 3 | `landing-cta rounded-xl` | `HeroSection.tsx`, `AuthNav.tsx` | Reuse | `ui/Button` primary (landing theme) |
| Button | Landing CTA secondary | 2 | `landing-cta-secondary` | `HeroSection.tsx`, `LandingSections.tsx` | Reuse | `ui/Button` secondary (landing) |
| Button | Auth submit (cyan) | 1 shared + 1 dupe | `bg-cyan-500 text-slate-950` | `AuthForm.tsx`, `reset-password` | Merge | `ui/Button` (auth theme) |
| Button | Secondary outline (slate) | ~7 / 5 files | `border-slate-300 bg-white` | `PurchaseConfirmDialog.tsx`, `me/esims` | Merge | `ui/Button` secondary |
| Button | Secondary outline (sky) | 2 | `border-sky-300 bg-sky-50\|white` | `DepositCta.tsx`, `DepositPendingBanner.tsx` | Merge | `ui/Button` secondary |
| Button | Link (sky) | ~15+ / ~10 files | `text-sky-700 hover:text-sky-800` | `DepositCta` link, `LocationDetail.tsx` | Merge | `ui/Button` / `ui/Link` |
| Button | Link (auth cyan) | 3 files | `text-cyan-700 hover:text-cyan-600` | login / register / forgot | Merge | `ui/Link` (auth) |
| Button | Compatibility CTA | 1 | `rounded-full bg-amber-500` | `CompatibilityButton.tsx` | Reuse | domain wrapper |
| Button | Ghost / close | 2 | `text-slate-500 hover:bg-slate-100` | Coverages + Compatibility modals | Merge | `ui/Button` ghost |
| Button | Icon avatar | 1 | `h-10 w-10 rounded-full bg-sky-700` | `UserMenu.tsx` | Reuse | Avatar / IconButton |
| Button | Tab (PlansStore) | 1 file | active `bg-sky-700`; idle hover slate | `PlansStore.tsx` | Replace | `ui/Tabs` |
| Button | Segment (LocationDetail) | 1 file | active `rounded-full bg-slate-900` | `LocationDetail.tsx` | Replace | `ui/Tabs` segmented |
| Button | List-row select | 1 | hover `border-sky-600 bg-sky-50` | `AndroidManufacturerPicker.tsx` | Reuse | interactive `ui/ListRow` |
| Button | Google Sign-In wrap | 1 | cyan focus; GIS owns chrome | `GoogleSignInButton.tsx` | Reuse | keep wrapper |
| Card | App panel | ~20+ / ~12 files | `rounded-2xl border-slate-200 bg-white p-6 shadow-sm` | deposit / eSIM pages | Merge | `ui/Card` |
| Card | List-row (hover) | 2 | `rounded-xl … hover:border-sky-300` | `LocationCard.tsx`, `me/esims` | Merge | `ui/ListRow` |
| Card | List-row (static) | 1 | `rounded-xl … shadow-sm` | `PackageRow.tsx` | Merge | `ui/ListRow` |
| Card | Plan / catalog | 1 | `rounded-xl … p-5 shadow-sm` | `PlanCard.tsx` | Reuse | `ui/Card` dense |
| Card | Landing | 1 file | `landing-card` | `FeaturedPlans.tsx` | Reuse | landing Card |
| Card | Auth glass | 1 shell | `rounded-3xl bg-white/95 backdrop-blur` | `AuthForm` AuthShell | Reuse | auth Card |
| Card | Nested inset | 1 | `bg-slate-50/80 border-slate-100` | `CexDepositForm.tsx` | Reuse | `ui/Card` muted |
| Card | Dropdown / menu | 2 | `shadow-lg` popover | `UserMenu.tsx`, `LocationSearch.tsx` | Reuse | Menu / Popover |
| Card | Dialog sheet | 3 | `rounded-t-2xl sm:rounded-2xl shadow-xl` | PurchaseConfirm, modals | Merge | `ui/Dialog` |
| Badge | eSIM status pill | 1 | `rounded-full bg-slate-100 text-xs` | `me/esims/page.tsx` | Reuse | `ui/Badge` |
| Badge | Setup step pills | 1 file | sky-700 / sky-100 / slate-200 | `me/esims/[id]/setup` | Replace | Stepper / Badge |
| Badge | Network (amber) | 1 | `border-amber-200 bg-amber-50` | `CexDepositForm.tsx` | Reuse | `ui/Badge` warning |
| Badge | Balance chip | 1 component | `rounded-full border shadow-sm` | `BalanceChip.tsx` | Reuse | keep `BalanceChip` |
| Badge | Landing step accent | 1 | `landing-accent rounded-full` | `LandingSections.tsx` | Reuse | landing only |
| Input | Auth field (cyan) | shared → auth forms | `rounded-lg … ring-cyan-500` | `AuthForm.tsx`, `PasswordField.tsx` | Merge | `ui/Input` auth |
| Input | App field (sky xl) | 3 | `rounded-xl … ring-sky-600` | deposit, voucher, CEX | Merge | `ui/Input` default |
| Input | Textarea (note) | 1 | `rounded-lg … ring-sky-200` | `EsimNoteForm.tsx` | Merge | `ui/Textarea` |
| Input | Search pill | 1 | `rounded-full … ring-sky-100` | `LocationSearch.tsx` | Reuse | `ui/Input` search |
| Input | Modal search | 2 | `bg-slate-50` + sky/amber ring | Coverages / Compatibility | Merge | `ui/Input` |
| Input | Checkbox (auth) | 1 | `text-cyan-600 ring-cyan-500` | `AuthForm.tsx` | Reuse | `ui/Checkbox` |
| Empty | Centered empty card | 4 | `rounded-2xl … p-8 text-center` | PlansStore, eSIMs, LocationDetail, deposit | Merge | `ui/EmptyState` |
| Empty | Inline / dropdown | 2 | plain text | `LocationSearch`, eSIM detail | Merge | `ui/EmptyState` inline |
| Alert | Page error (amber panel) | ~7 / 6 files | `rounded-2xl amber-50 p-6` | PlansStore, eSIM, deposit | Merge | `ui/Alert` warning |
| Alert | Inline amber box | ~8 / 5 files | `rounded-xl amber-50` | Wallet/Cex, eSIM, LocationDetail | Merge | `ui/Alert` warning |
| Alert | Sky info / banner | ~6 / 4 files | `border-sky-200 bg-sky-50` | `DepositPendingBanner`, deposit strip | Merge | `ui/Alert` info |
| Alert | Network warning | 1 | `rounded-2xl amber-50 p-5` | `DepositNetworkWarning.tsx` | Merge | `ui/Alert` warning |
| Alert | Success (emerald) | ~3 | `emerald-50` | EsimNote, voucher | Merge | `ui/Alert` success |
| Alert | Auth error (red) | 3 | `red-50 text-red-800` | AuthForm, set/reset-password | Merge | `ui/Alert` danger |
| Alert | Voucher error (rose) | 1 | `rose-50` | `VoucherRedeemForm.tsx` | Merge | `ui/Alert` danger |
| Alert | Plain text error | 1 | no box | `EsimNoteForm.tsx` | Replace | `ui/Alert` |
| Skeleton | List / detail pulse | 2 exports | `animate-pulse bg-slate-200/80` | `ui/ListSkeleton.tsx` | Reuse | `ui/Skeleton` |
| Skeleton | Deposit page | 1 | duplicated Pulse | `DepositSkeleton.tsx` | Merge | `ui/Skeleton` recipe |
| Skeleton | Header / chip | 2 | pulse rounded-full | `AccountCluster`, `BalanceChip` | Reuse | `ui/Skeleton` circle |
| Skeleton | Price placeholder | 1 | pulse text | `CatalogPriceDisplay.tsx` | Reuse | `ui/Skeleton` text |
| Skeleton | Spinner | 2 | `animate-spin` | AuthForm, PurchaseConfirm | Merge | `ui/Spinner` |

## Cap2 implication

Introduce thin theme-bound primitives (not brand-direct per ADR 016):

- `ui/Button` (primary / secondary / ghost / link)
- `ui/Card` + `ui/ListRow`
- `ui/Badge`
- `ui/Input` / `ui/Textarea`
- `ui/Alert`
- `ui/EmptyState`
- `ui/Skeleton` / `ui/Spinner`

Tone via **theme** (app sky vs auth cyan vs landing CSS vars). Keep domain wrappers (`DepositCta`, `BalanceChip`, Google, Compatibility) as composition.

**Out of Cap2:** Tabs/Stepper Replace rows (may stay deferred or a thin follow-up), AppShell dark shell (Cap3), iconography (Cap5).

## Summary counts and risk

Risk = expected Cap2 conflict / blast radius (not calendar estimate).

| Component | Variants | Merge | Reuse | Replace | Risk |
|-----------|---------:|------:|------:|--------:|------|
| Button | 16 | 9 | 5 | 2 | **High** |
| Card | 9 | 4 | 5 | 0 | Medium |
| Badge | 5 | 0 | 4 | 1 | Low |
| Input | 6 | 4 | 2 | 0 | Medium |
| Empty | 2 | 2 | 0 | 0 | Low |
| Alert | 8 | 7 | 0 | 1 | Medium |
| Skeleton | 5 | 2 | 3 | 0 | Low |

## Cap2 migration order

Take highest return / conflict surface first; stabilize before cards:

1. **Button** (16 variants) — largest payoff
2. **Alert** — many duplicates + semantics (warning / info / success / danger)
3. **Input** — auth + app can share one primitive (theme tone)
4. **Card** — after Button/Input stable
5. **Skeleton**
6. **Empty**
7. **Badge** — smallest impact

## Cap2 done criteria

**Cap2 is complete when:**

- every **Merge** priority has one canonical UI primitive,
- new development uses only canonical primitives (no new ad-hoc Button/Card/Input/Alert class stacks),
- **Replace** rows are no longer referenced as Buttons/Cards (or explicitly deferred with a follow-up note),
- there are no new duplicated style variants for Button / Card / Input / Alert.

Cap2 remains architecture-only: **no intentional visual change** (same stop rule as Cap1). Visual redesign is Cap3+.

**Closed:** Cap2 Review quality gate + API Freeze — see [status.md](./status.md). Remaining leftovers are migration backlog, not Cap2 reopen. Cap3 must not change Cap2 APIs.
