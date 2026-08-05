# Cap5 — Design Consistency Polish Implementation Plan

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-05 |
| Prerequisite | [Design Lock](./cap5-design-consistency-lock.md) = **Accepted** |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), [capability-ledger.md](./capability-ledger.md) |
| Repo | `roamkit-web` (code), this doc in `roamkit-docs` |

**Discipline:** One slice = one goal. Design Lock is **closed** — no new design decisions.  
**Cap2 API Freeze holds.** No new primitives, tokens, or layouts until Cap5 closes.

```text
Design Lock ✅ Accepted
  → Implementation Plan ✅ Accepted (this doc)
    → Cap5.1 Tabs (pilot) → Pilot Freeze → Cap5.2 → Cap5.3 → Cap5.4
```

**Cap5.1 may start.** Do not reopen Design Lock decisions.

---

## Whole-capability acceptance

Cap5 is done when all Design Lock stop rules hold, including:

- Cap5 backlog chrome (tabs, stepper, avatar) bound to existing `--app-*`
- Cap3.5 `SKY700_ALLOWLIST` cleared for Cap5-scoped files
- Cap2 public APIs unchanged; no new `ui/*` primitives
- No new tokens / layouts / features
- Landing + Auth untouched
- Chrome consistency matrix green
- Legacy Chrome Audit green on Cap5 routes

---

## Quality gates (not features)

### 1. Migration Matrix

| Element | Surfaces | Old | New | Smoke | Status |
|---------|----------|-----|-----|-------|--------|
| Tabs | `PlansStore` (`/plans`); `LocationDetail` service tabs | ⏳ | ⏳ | ⏳ | **Pilot** Cap5.1 |
| Stepper | `/me/esims/[id]/setup` pills | ⏳ | ⏳ | ⏳ | Cap5.2 |
| Avatar | `UserMenu` trigger | ⏳ | ⏳ | ⏳ | Cap5.3 |

Legend: Old = pre-Cap5 baseline · New = `--app-*` chrome · Smoke = staging green.

Isolation (must stay unchanged):

| Route | Role |
|-------|------|
| `/` | Landing |
| `/login` (or any AuthShell route) | Auth Cap4 |
| AppShell layout / TopBar / nav IA | Cap3 |

### 2. Golden Route + Pilot Freeze

> **Golden Route (pilot):** `/plans` — `PlansStore` tabs  
> Secondary in same slice: `LocationDetail` service tabs (theme only; same Cap5.1 PR)

```text
Cap5.1  Tabs (pilot)
  ↓
Pilot Freeze  (/plans staging smoke; LocationDetail spot-check)
  ↓
Cap5.2  Stepper
  ↓
Cap5.3  Avatar
  ↓
Cap5.4  Validation + close
```

> **Cap5.2 (Stepper) does not start until the Tabs pilot is confirmed on staging with no visual regressions.**  
> If `/plans` tab chrome staging smoke is not green, Cap5.2 does not open.  
> Stepper and avatar must not be touched until tabs are visually confirmed.

### 3. Acceptance gate (leave Cap5.1)

Before Cap5.2:

- [ ] Tab API / navigation / state unchanged (`tab` params, roles, labels)
- [ ] Stepper + UserMenu **not** in Cap5.1 diff
- [ ] No Landing / Auth / AppShell layout files in Cap5.1 diff
- [ ] Cap2 public API unchanged; no new primitives/tokens
- [ ] `/plans` staging smoke green (desktop + mobile); LocationDetail service tabs spot-check

### 4. Visual Diff (pilot — required)

For `/plans` tabs (manual; no automated screenshot suite required):

| Artifact | Required |
|----------|----------|
| Before screenshot | Baseline tab chrome |
| After screenshot | Cap5.1 `--app-*` binding |
| Viewports | **desktop**, **tablet**, **mobile** |
| Expected diffs note | Intentional theme only; flag anything else as regression |

### 5. Exit criteria — Cap5 Complete

```text
Cap5 Complete
```

when **all** are true:

- Migration Matrix all three elements New + Smoke ✓
- Chrome consistency matrix green
- Legacy Chrome Audit green
- Cap2 API Freeze held
- Landing + Auth still untouched
- Close note in [status.md](./status.md)

---

## Work order (strict)

```text
Cap5.1  Tabs (pilot: PlansStore + LocationDetail service tabs)
  ↓
Pilot Freeze
  ↓
Cap5.2  Stepper pills
  ↓
Cap5.3  Avatar trigger chrome
  ↓
Cap5.4  Validation + close Cap5
```

| Slice | Goal | Must not |
|-------|------|----------|
| Cap5.1 | Bind tab active/idle/hover chrome to `--app-*` | Change tab API/nav/state; touch stepper/avatar; new `ui/Tabs` |
| Cap5.2 | Bind stepper active/completed/inactive to `--app-*` | Change step logic/copy/flow; touch tabs/avatar |
| Cap5.3 | Bind avatar trigger surface/border/radius/focus to `--app-*` | Redesign dropdown menu panel/behaviour; touch tabs/stepper |
| Cap5.4 | Matrix + Legacy Chrome Audit; clear Cap3.5 allowlist; close Cap5 | New design decisions |

---

## Cap5.1 — Tabs (pilot)

**Golden Route:** `/plans` (`PlansStore` tablist).  
**Same PR:** `LocationDetail` service tablist (underline chrome on elevated) — theme only.

### In scope

| Area | Notes |
|------|--------|
| `components/PlansStore.tsx` | Active `bg-sky-700` → `--app-primary*` (or equivalent `--app-*`); idle/hover on shell |
| `components/LocationDetail.tsx` `ServiceTab` | Slate underline active/idle → `--app-*` / elevated-consistent chrome |
| Focus | Existing focus patterns → `--app-focus-ring` where applicable |

### Out of scope

- Tab count, labels, `searchParams` / filter semantics
- `SegmentButton` data-amount filter (not Cap3.5 allowlist; Cap6 unless Plan later expands — **leave**)
- Stepper, UserMenu, PlanCard, Deposit
- New `ui/Tabs` primitive

### Cap5.1 done when

- [ ] PlansStore + LocationDetail service tabs use `--app-*` (no Cap5-scoped `sky-700` on tabs)
- [ ] Behaviour / a11y roles unchanged
- [ ] Staging Pilot Freeze green on `/plans`
- [ ] Cap5.2 still closed

---

## Cap5.2 — Stepper pills

**Open only after Pilot Freeze.**

**Route:** `/me/esims/[id]/setup`

### In scope

| Area | Notes |
|------|--------|
| Setup step `<ol>` pills | `active` / `completed` / `inactive` → `--app-*` |
| Contrast | AA on shell / elevated context as today |

### Out of scope

- Step machine, step count, setup copy, install guides
- `text-sky-700` helper links on the same page (Cap6 audit — not Cap5 invent)
- Tabs, avatar

### Cap5.2 done when

- [ ] Three visual states token-bound
- [ ] No step logic change
- [ ] Spot-check setup route on staging

---

## Cap5.3 — Avatar trigger chrome

**Open only after Cap5.2.**

### In scope

| Area | Notes |
|------|--------|
| `UserMenu` circular trigger | Fill / hover / border / radius / `focus-visible` ring + `--app-background` offset → `--app-*` |

### Out of scope

- Menu open/close, items, logout, panel layout/styles (leave panel alone)
- New Avatar primitive
- AccountCluster structure

### Cap5.3 done when

- [ ] Trigger chrome on `--app-*` (no `sky-700` / `sky-500` on avatar button)
- [ ] Dropdown behaviour unchanged
- [ ] Spot-check TopBar account menu on staging

---

## Cap5.4 — Validation + close

**Goal:** Close Cap5. No design invention.

### Chrome consistency matrix

| Element | Result |
|---------|--------|
| Button | ☐ stays ✓ |
| Input | ☐ stays ✓ |
| Alert | ☐ stays ✓ |
| Card | ☐ stays ✓ |
| Tabs | ☐ Cap5.1 |
| Stepper | ☐ Cap5.2 |
| Avatar | ☐ Cap5.3 |

### Legacy Chrome Audit (Cap5-scoped)

On Cap5 surfaces (`/plans`, LocationDetail tabs, setup stepper, UserMenu trigger), confirm **no**:

- [ ] Hardcoded `sky-*` that were in Cap5 scope (Cap3.5 allowlist rows)
- [ ] Old border/radius chrome for tabs / stepper / avatar that bypasses `--app-*`
- [ ] Local chrome styles that bypass Cap2 primitives + `--app-*` for those three elements

Cap5.4 close table (must all read Cleared):

| Legacy element | Status |
|----------------|--------|
| Tabs | ☐ Cleared |
| Stepper | ☐ Cleared |
| Avatar | ☐ Cleared |
| Cap3.5 sky allowlist | ☐ Cleared |

Cap3.5 `SKY700_ALLOWLIST` must be **empty** (or removed) after Cap5.4 — those three files no longer exempt.

### Isolation smoke

| Route | Check |
|-------|--------|
| `/` | Landing unchanged |
| `/login` | Auth unchanged |
| `/plans` | Tabs green; AppShell layout unchanged |
| `/me/esims/[id]/setup` | Stepper green |
| Authenticated TopBar | Avatar trigger green |

### Cap5.4 done when

- Cap5 stop rule satisfied
- Capability marked **CLOSED** in status
- Cap6 may open (audit only — no Cap5 reopen)

---

## Explicit non-goals (every slice)

- Cap1–Cap4 reopen
- Landing / Auth / AppShell layout
- PlanCard, Deposit UX, Dialog, Popover redesign
- Full-repo `sky-*` / spacing / radius / shadow purge (**Cap6**)
- New npm dependencies
- New Cap2 variants / sizes / tones / primitives
- New CSS theme tokens

---

## PR hygiene

- Branch from `develop`; PR into `develop`
- Conventional Commits: `feat(ui): Cap5.N …` / `docs(design): …`
- One slice per PR
- Squash-merge when CI green
- Cap5.1 PR description must list **Pilot Freeze** checklist for `/plans`

---

## Status

| State | Meaning |
|-------|---------|
| Draft — awaiting acceptance | Closed |
| **Accepted** | **Current** — Cap5.1 may start |
| Done | Cap5.4 closed Cap5 |

**Next:** Cap5.1 Tabs pilot (Golden Route `/plans`) → Pilot Freeze → Cap5.2 → Cap5.3 → Cap5.4.
