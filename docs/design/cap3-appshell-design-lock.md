# Cap3 — AppShell Variant A Design Lock

| Field | Value |
|-------|-------|
| Status | **Draft — awaiting acceptance** |
| Date | 2026-08 |
| Capability | Cap3 — AppShell Variant A (visual chrome only) |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), Cap2 Review (`roamkit-web/components/ui/CAP2_REVIEW.md`) |
| Pattern | Same as Landing PR2: **Design Lock → Implementation Plan → code** |

This document locks Cap3 decisions so implementation does not wander.  
**Do not write Cap3 code until this lock is Accepted.**  
**Do not start the Implementation Plan until this lock is Accepted.**

Cap2 remains **CLOSED**. Cap3 must not reopen Cap2 except for a true visual regression.

---

## One-sentence visual goal

> **Dark shell + light elevated surfaces.**

Nothing else. Not “modern”. Not “prettier”. This sentence is the acceptance lens for every Cap3 PR.

---

## Guardrails (locked)

| Gate | Lock |
|------|------|
| **Shell scope** | background, TopBar, nav chrome, shell spacing, surfaces, cards-as-chrome placement — **not** PlanCard, deposit banners, dialogs, popovers, tabs, or marketing pages |
| **Vizualni cilj** | Dark shell + light elevated surfaces |
| **Surface model** | Background → Surface → Elevated Surface (**max 3 token levels**) |
| **CTA** | Store primary `sky-700` → brand cyan **only** via `--app-*` theme aliases |
| **Layout** | No DOM hierarchy change; no navigation structure change |
| **Primitive policy** | Cap2 primitives only; no new local Button / Card / Input style systems |
| **Theme policy** | Only `--app-*` / theme aliases in Cap3 chrome; **no** direct brand-token use in components |
| **Performance** | No new JS dependencies; CSS-only where possible |
| **Stop rule** | AppShell on Variant A theme; Cap2 API untouched; layout structure same; landing + auth untouched |

---

## Acceptance criteria (locked)

### 1. Canonical chrome only

Cap3 may change **only** chrome:

- page / shell background
- TopBar
- navigation chrome (visual only)
- surfaces
- card **placement** on surfaces (using existing `ui/Card` / `ListRow`)

Cap3 must **not** change page content, copy, domain UI, or feature behaviour.

### 2. Zero feature work

Cap3 must not include:

- new product features
- new API calls
- new hooks
- new navigation destinations or IA

This is a **pure visual** capability.

---

## What is “shell” (in / out)

### In scope

| Area | Notes |
|------|--------|
| `AppShell` | Outer frame: background, padding, max-width, content gap |
| `TopBar` | Existing grid chrome (left nav slot / right account slot) |
| Navigation chrome | Visual treatment of existing `nav` / `rightSlot` only |
| Shell spacing | Horizontal padding, vertical page padding, header→content gap — **owned by AppShell** |
| Surfaces | Restyle `--app-*` for Variant A; wire chrome to aliases |
| Cards as surfaces | Elevated light panels on dark shell via Cap2 `ui/Card` / existing card chrome |

**AppShell routes (full surface — 6):**

| Route | Component |
|-------|-----------|
| `/plans` | `PlansStore` |
| `/[slug]-esim` | `LocationDetail` |
| `/me/esims` | list |
| `/me/esims/[id]` | detail |
| `/me/esims/[id]/setup` | setup |
| `/me/deposit` | deposit |

### Explicitly out of scope

| Area | Why |
|------|-----|
| Marketing landing (`/`) | Cap1/PR2; Cap3 does not touch |
| AuthShell / auth pages | Cap4 |
| PlanCard density / catalog tile redesign | Cap2 Reuse / backlog |
| Deposit banners, wallet/CEX panels, voucher chrome | Domain; Cap2 intentional exceptions |
| Dialogs, popovers, dropdowns, tabs, steppers | Not shell |
| Cap2 migration backlog (raw Button/Input leftovers) | Separate small PRs; not Cap3 |
| Cap2 public API changes | API Freeze — never drive-by in Cap3 |
| New primitives | Cap2 closed; Cap5+ if needed |
| Iconography / illustrations | Cap5 |
| Light/dark theme toggle | Not Cap3 (ADR 016) |

---

## Surface model (locked — max 3)

Semantic theme levels only:

```text
Background          (--app-background)     dark shell canvas
  ↓
Surface             (--app-surface)        default panel / content well
  ↓
Elevated Surface    (--app-surface-elevated) light cards / raised chrome
```

**Rules:**

- Do **not** invent a fourth token level (no `--app-shell`, `--app-page`, `--app-section` as color layers).
- Composition (TopBar, page content, Card, section) **maps onto** these three — it does not add token tiers.
- Cards remain Cap2 containers; Cap3 places them on Elevated Surface and ensures contrast against Background.
- No skipping for identity (e.g. full-page elevated white pretending to be the shell).

### Token direction (values chosen in Implementation Plan)

| Alias | Cap3 direction | Cap1 today (baseline) |
|-------|----------------|------------------------|
| `--app-background` | Dark shell (brand-adjacent ink / surface family) | `#f8fafc` (slate-50) |
| `--app-surface` | Mid / content well as needed for hierarchy | `#ffffff` |
| `--app-surface-elevated` | **Light** elevated cards | `#ffffff` |
| `--app-text` / `--app-text-muted` | Readable on dark shell **and** on light cards (context-appropriate) | slate-900 / slate-600 |
| `--app-primary` | **Brand cyan** (`var(--color-primary)`) | sky-700 temporary |
| `--app-primary-hover` | Brand primary hover | sky-800 |
| `--app-border` | Compatible with dark shell + light cards | slate-200 |

Exact hex / contrast pairs are fixed in the **Implementation Plan**, not reopened as new goals. Contrast must meet WCAG AA for text on both shell and elevated cards.

---

## CTA migration (locked)

```text
sky-700 (store primary today)
  ↓
brand cyan
```

**Only** through `--app-primary` / `--app-primary-hover` (and Cap2 primitives / chrome that consume app theme).

Forbidden:

- Hand-scattered `bg-cyan-500` / `bg-sky-700` replacements as Cap3 “design”
- Direct brand token references in components (`var(--color-primary)` in TSX/CSS of pages)
- Changing Button/Card/Input **public props** to force cyan

Allowed:

- Restyle `--app-*` values
- Bind AppShell chrome (and Cap2 `tone="app"` **internals**) to theme aliases without API changes

---

## Layout lock

Baseline today (`AppShell`):

```text
min-h-screen bg-slate-50 px-6 py-16 text-slate-900
  main max-w-{2xl|3xl|4xl}
    TopBar (grid 1fr / auto)
    page-content mt-8
      children
```

Cap3 may change **visual tokens and spacing values** owned by the shell.

Cap3 must **not**:

- Change DOM order or TopBar grid contract (`nav` left, `rightSlot` right)
- Add sidebars, bottom tabs, or new nav IA
- Move account chrome out of `rightSlot`
- Fork a second AppShell for “dark” vs “light” pages

AuthShell already approximates dark frame + light card — Cap3 brings that **hierarchy** to store/account; it does not merge AuthShell into AppShell.

---

## Primitive & theme policy

1. **Cap2 API Freeze holds.** Cap3 PRs must not edit Cap2 public APIs. Internal theme binding without prop changes is allowed when required for CTA/surface.
2. **No parallel style systems.** No new `app-button`, `shell-card`, or one-off CTA classes that bypass `ui/*`.
3. **Theme only.** Cap3 chrome consumes `--app-*`. Brand tokens stay behind theme aliases (ADR 016).
4. **Migration leftovers** from Cap2 Review stay backlog — do not fold into Cap3 slices.

---

## Performance

- No new npm / client JS dependencies for Cap3.
- Prefer CSS variables + existing Tailwind / utilities.
- Motion (if any): respect `prefers-reduced-motion`; do not gate navigation or CTAs on animation.

---

## Proposed implementation slices (outline only)

Not started. Locked as the **preferred PR shape** after Implementation Plan:

```text
Cap3.1  Shell tokens + background
Cap3.2  TopBar + navigation chrome
Cap3.3  Surface migration (elevated cards on dark shell)
Cap3.4  CTA theme migration (sky → cyan via --app-*)
Cap3.5  Validation + consistency (all 6 AppShell routes)
```

Pilot preference (to confirm in Implementation Plan): `/plans` + `/me/esims` (+ one detail) before full route sweep.

---

## Stop rule

Cap3 is **CLOSED** when all are true:

1. All six AppShell routes use Variant A chrome (dark shell + light elevated surfaces).
2. Store primary CTAs that are in shell/theme scope resolve through `--app-primary` (cyan), not ad-hoc sky for that chrome path.
3. Cap2 primitive **APIs** unchanged (Freeze held).
4. DOM / nav structure unchanged.
5. Landing and auth **untouched**.
6. No new features, API calls, hooks, or nav destinations.
7. Staging smoke on the six routes passes (desktop + mobile breakpoints).

Anything beyond this is Cap4+ or a separate backlog PR.

---

## Explicitly unchanged (do not reopen in Cap3)

- Cap2 primitive public APIs and composition rules
- Cap2 Review intentional exceptions (PlanCard, deposit domain banners, etc.)
- Landing marketing Design Lock / PR2
- Auth flows and AuthShell (Cap4)
- Product copy, pricing, deposit logic, eSIM flows

---

## Entry checklist

Before first Cap3 implementation PR:

- [ ] This Design Lock **Accepted** (status flipped below)
- [ ] Implementation Plan written (token hex + Cap3.1–3.5 detail + pilot routes)
- [ ] Explicit: zero Cap2 API edits in Cap3 PRs
- [ ] Explicit: landing + auth out of diff

---

## Status

| State | Meaning |
|-------|---------|
| **Draft — awaiting acceptance** | Current |
| Accepted | Safe to write Implementation Plan |
| Superseded | Only via explicit redesign decision (new lock) |

**Acceptance:** reply GO / Accepted on this document (or merge PR with status → Accepted).  
**Next after Accepted:** Cap3 Implementation Plan (still no code).  
**Next after Implementation Plan Accepted:** Cap3.1 first PR.
