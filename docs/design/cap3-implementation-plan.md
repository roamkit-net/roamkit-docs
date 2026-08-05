# Cap3 — AppShell Variant A Implementation Plan

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Prerequisite | [Design Lock](./cap3-appshell-design-lock.md) = **Accepted** |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md) |
| Repo | `roamkit-web` (code), this doc in `roamkit-docs` |

**Discipline:** One slice = one goal. Design Lock is **closed** — no new design decisions.  
**Cap2 API Freeze holds.** No new primitives, tokens, AppShell variants, or colors until Cap3 closes.

```text
Design Lock ✅ Accepted
  → Implementation Plan ✅ Accepted (this doc)
    → Cap3.1 → Cap3.2 → Cap3.3 (pilot) → smoke → Cap3.3b → Cap3.4 → Cap3.5
```

---

## Whole-capability acceptance

Cap3 is done when all Design Lock stop rules hold, including:

- Six AppShell routes on Variant A chrome
- Cap2 public APIs unchanged
- Landing + auth untouched
- Zero feature work (no new APIs, hooks, nav destinations)
- No new JS dependencies

---

## Quality gates (not features)

### 1. Migration Matrix

Track every AppShell surface. Update after each slice.

| Route | Old | New | Smoke | Status |
|-------|-----|-----|-------|--------|
| `/plans` | ✓ | ⏳ | ⏳ | pending |
| `/me/esims` | ✓ | ⏳ | ⏳ | **Golden Route** (Cap3.3a) |
| `/me/esims/[id]` | ✓ | ⏳ | ⏳ | pending |
| `/me/esims/[id]/setup` | ✓ | ⏳ | ⏳ | pending |
| `/me/deposit` | ✓ | ⏳ | ⏳ | pending |
| `/[slug]-esim` | ✓ | ⏳ | ⏳ | pending |

Legend: Old = pre-Cap3 baseline still reachable · New = Variant A chrome applied · Smoke = staged check green.

### 2. Rollback point

> **If pilot smoke is not green, migration stops.**  
> Do not open Cap3.3b (propagate) or Cap3.4 until the pilot gate passes.

Same rule between Cap3.1 → Cap3.2 and Cap3.2 → Cap3.3 if shell/TopBar smoke fails.

### 3. Acceptance gate (before leaving Cap3.1 / Cap3.2 for surfaces)

Before Cap3.3 propagate (and before Cap3.4):

- [ ] Same spacing SoT (AppShell-owned)
- [ ] Same shell chrome
- [ ] Same TopBar contract
- [ ] No local layout overrides for basic padding/max-width
- [ ] Cap2 public API unchanged (`git diff` on `ui/*` props/types)

### 4. Visual Diff (pilot routes — required)

For `/me/esims` pilot (and each route as it migrates):

| Artifact | Required |
|----------|----------|
| Before screenshot | Baseline (light slate shell) |
| After screenshot | Variant A (dark shell + light elevated) |
| Expected diffs note | Planned redesign only — list intentional changes; flag anything else as regression |

Not for redesign debate — for regression control.

### 5. Exit criteria — Cap3 Complete

```text
Cap3 Complete
```

when **all** are true:

- All **6** AppShell surfaces use the same AppShell
- No layout hacks / local padding systems for shell basics
- Cap2 API Freeze still held
- Migration Matrix all New + Smoke ✓
- Cap3.5 smoke matrix green (desktop + mobile)

---

## Work order (strict)

```text
Cap3.1  Shell tokens + background
  ↓
Cap3.2  TopBar + navigation chrome
  ↓
Cap3.3  Surface migration (pilot → then rest)
  ↓
Cap3.4  CTA theme migration
  ↓
Cap3.5  Validation + consistency
```

| Slice | Goal | Must not |
|-------|------|----------|
| Cap3.1 | Restyle `--app-*` for Variant A; wire `AppShell` background/text frame | Restyle TopBar, cards, or Button |
| Cap3.2 | TopBar + nav/rightSlot chrome on dark shell | Migrate page cards / PlanCard |
| Cap3.3 | Elevated surfaces on **one pilot route**, then propagate | CTA sky→cyan; Cap2 API edits |
| Cap3.4 | Store primary via `--app-primary` (cyan) | Domain deposit banners; landing/auth |
| Cap3.5 | Smoke all six routes; close Cap3 | New design decisions |

---

## Locked token values (Cap3.1)

Restyle **only** the App theme block in `roamkit-web/app/globals.css`. Brand tokens stay source of truth; app aliases reference brand where possible.

| Alias | Cap3 value | Notes |
|-------|------------|--------|
| `--app-background` | `var(--color-background)` (`#05070a`) | Dark shell canvas |
| `--app-surface` | `var(--color-surface)` (`#0d1117`) | Mid well (may match chrome strips; not a 4th level) |
| `--app-surface-elevated` | `#ffffff` | Light elevated cards |
| `--app-text` | `#0f172a` | Text **on elevated / light** surfaces |
| `--app-text-muted` | `#475569` | Muted on light surfaces |
| `--app-chrome-text` | `var(--color-text)` (`#f0f4f8`) | Text on dark shell (TopBar / chrome) — text alias, **not** a 4th surface |
| `--app-chrome-text-muted` | `var(--color-text-muted)` (`#94a3b8`) | Muted chrome |
| `--app-primary` | `var(--color-primary)` | Brand cyan (wired in Cap3.4 for CTAs) |
| `--app-primary-hover` | `var(--color-primary-hover)` | |
| `--app-primary-foreground` | `var(--color-primary-foreground)` | |
| `--app-focus-ring` | `color-mix(… primary …)` | Keep pattern; track primary |
| `--app-border` | `#e2e8f0` | Borders on elevated cards |
| `--app-border-chrome` | `rgba(148, 163, 184, 0.18)` | Optional hairline on dark shell |

Do **not** introduce `--app-shell` as a fourth surface color. If TopBar needs a subtle strip, use `--app-surface` (dark mid) or transparent over `--app-background`.

**Contrast:** AA for chrome text on `--app-background` and body text on `--app-surface-elevated`.

Spacing SoT — add Cap3.1 aliases (same pixel values as today; no rhythm redesign):

| Alias | Value |
|-------|--------|
| `--app-gutter-x` | `1.5rem` |
| `--app-page-padding-y` | `4rem` |
| `--app-header-gap` | `2rem` |
| `--app-content-max` | `56rem` |
| `--app-content-max-mid` | `48rem` |
| `--app-content-max-narrow` | `42rem` |

**Cap3.1 wiring:** `AppShell` outer frame uses `--app-background` + chrome text + spacing tokens (via `.app-shell*` utilities). Do **not** migrate cards or Button in Cap3.1. **No DOM hierarchy change.**

Baseline to replace in `AppShell.tsx`:

```text
min-h-screen bg-slate-50 px-6 py-16 text-slate-900
```

---

## Cap3.1 — Shell tokens + background

**Files (expected):** `app/globals.css`, `components/AppShell.tsx`, `AppShell` tests if class assertions exist.

### Ownership (locked)

> **AppShell owns layout; pages own content.**

AppShell owns: width, spacing, shell background, TopBar slot, chrome text color.  
Pages own: what they render inside `page-content`. Never the reverse.

### Cap3.1 forbidden

- New design tokens beyond the locked Cap3.1 table (+ spacing SoT aliases below)
- New primitives
- Local layout helpers / page-level AppShell forks
- Inline layout spacing that bypasses `--app-*` (use spacing tokens)
- Cap2 API changes
- TopBar visual redesign beyond inheriting chrome text (Cap3.2)
- Card / Button migration (Cap3.3 / Cap3.4)

### Cap3.1 Definition of Done

- [ ] `AppShell` is the only layout entry point for the six store/account routes
- [ ] No new local layout wrappers introduced
- [ ] TopBar remains composition-only (no new primitives; Cap3.2 for contrast polish)
- [ ] Layout spacing uses `--app-*` spacing aliases (not ad-hoc `px-6` / `py-16` / `mt-8` literals in TSX)
- [ ] `--app-*` color values match the locked table
- [ ] All six AppShell routes show dark shell background (interim: page chrome text may sit on dark canvas until Cap3.3)
- [ ] Landing + auth unchanged
- [ ] Lint / tests green
- [ ] Cap2 public API untouched

### Layout Debt Register

Record “later” findings here — not in code comments. Delete table when empty after Cap3 Complete.

| Route | Debt | Planned capability |
|-------|------|--------------------|
| — | — | — |

### Smoke checklist (Cap3.1 — shell frame only)

| Route | Visual | Responsive | Keyboard | Dark shell |
|-------|--------|------------|----------|------------|
| `/plans` | ☐ | ☐ | ☐ | ☐ |
| `/me/esims` | ☐ | ☐ | ☐ | ☐ |
| `/me/esims/[id]` | ☐ | ☐ | ☐ | ☐ |
| `/me/esims/[id]/setup` | ☐ | ☐ | ☐ | ☐ |
| `/me/deposit` | ☐ | ☐ | ☐ | ☐ |
| `/[slug]-esim` | ☐ | ☐ | ☐ | ☐ |

**Out of Cap3.1 code:** TopBar retint beyond inheriting shell text; Card migration; Button CTA.

---

## Cap3.2 — TopBar + navigation chrome

TopBar is **part of the Shell**, not a page component.

### Ownership

> **TopBar owns navigation chrome.** Pages own content — not mini-headers that duplicate TopBar.

TopBar is the sole place for: back/nav slot, account actions, balance chip, future notifications.  
Pages must not add competing chrome headers for those roles.

### Cap3.2 locks (operational)

| Decision | Lock |
|----------|------|
| Sticky policy | **Static** (not sticky) — one policy for all AppShell routes |
| Scroll behaviour | **Always visible** (follows static; no hide-on-scroll, no scroll shadow) |
| Height SoT | `--app-topbar-height` (single source) |
| Safe area | Shell/`TopBar` respect `env(safe-area-inset-*)` |
| Cap2 | No new primitives; use existing cluster / Button where needed |

### Cap3.2 forbidden

- Wallet balance redesign, search, notifications, theme switch, UserMenu refactor
- Sticky on some routes only
- New AppShell forks / page-level mini-headers
- Cap2 API changes; Card/CTA migration (Cap3.3 / Cap3.4)

### Cap3.2 Definition of Done

- [ ] TopBar uses `--app-topbar-height`, chrome border, shell nav link contrast
- [ ] Static + always-visible scroll policy applied uniformly
- [ ] Safe-area insets on shell frame
- [ ] App `AuthNav` / account cluster readable on dark shell (no product redesign)
- [ ] Grid contract unchanged (`nav` left, `rightSlot` right)
- [ ] Landing AuthNav `variant="landing"` untouched
- [ ] Cap2 API Freeze held; lint/tests green

**Files (expected):** `globals.css`, `TopBar.tsx`, shell `nav` link classes on AppShell callers, `AuthNav` app focus/chrome only if required.

**Out:** Page body cards; PlanCard; deposit panels; Cap3.3 surface work.

### Pilot Readiness (gate before Cap3.3)

Do **not** open Cap3.3a until all are true:

- [ ] Cap3.1 shell done
- [ ] Cap3.2 TopBar done
- [ ] Spacing SoT in place
- [ ] Shell/TopBar contrast resolved (chrome text on dark)

Then pilot `/me/esims` only.

### Pilot Scorecard

Filled during Cap3.3a smoke — see **Cap3.3a Pilot Scorecard** below (do not maintain a second copy here).

---

## Cap3.3 — Surface migration

**Goal:** Light elevated surfaces on dark shell using Cap2 `ui/Card` / existing elevated chrome — **no Cap2 API changes**.

### Pilot rule (acceptance — required)

> **One route is the pilot.** After the pilot is accepted and staging smoke passes, apply the same pattern to the remaining AppShell routes.

Do **not** propagate Variant A surfaces to all six routes in the same step as the first visual call.  
Do **not** parallelize two pilot routes.

| Step | Route | Gate |
|------|-------|------|
| Cap3.3a Pilot | **`/me/esims`** | Visual accept + staging smoke → Golden Route |
| Cap3.3b Propagate | `/plans`, `/[slug]-esim`, `/me/esims/[id]`, `/me/esims/[id]/setup`, `/me/deposit` | Match Golden Route pattern only |

**Why `/me/esims`:** Account shell with Card / Skeleton / Empty / ListRow — exercises elevated surfaces without PlanCard catalog density.

### Cap3.3a — Golden Route (`/me/esims`)

Treat Cap3.3a as the **reference AppShell implementation**, not “the first of six similar PRs”.

> **Golden Route:** After pilot smoke is green, `/me/esims` is frozen as the composition reference. Later routes copy it; they do not invent a second pattern.

#### Composition pattern (locked for propagate)

| Layer | On `/me/esims` |
|-------|----------------|
| Shell canvas | `AppShell` / `--app-background` + chrome text |
| Page header on shell | `AppPageHeader` slots use `--app-chrome-text` / `--app-chrome-text-muted` |
| Elevated content | Cap2 `Card` / `ListRow` / `ListSkeleton` / `Alert` (light panels) |
| Actions | Existing Cap2 `Button` / `DepositCta` — **no** Cap2 API edits; CTA cyan = Cap3.4 |

#### Cap3.3a operational rules

1. **Migrate layout / contrast — do not beautify Cap2.** If Button / Card / Badge / Input look imperfect, open a debt row — do not “quick polish” primitives.
2. **Visual Debt Register** (below) for leftover margins, local wrappers, interim contrast — do not leave as “fix later in chat”.
3. **Smoke is end-to-end**, not “looks fine”: nav, scroll, loading, empty, error, keyboard, mobile (~390), Fold/narrow, balance chip, user menu.
4. **Pilot Freeze:** when smoke passes, **do not keep editing `/me/esims`** except true regressions. Propagate next.

#### Cap3.3a Definition of Done

- [ ] `/me/esims` header readable on dark shell (chrome text tokens)
- [ ] List / empty / loading / error sit on light elevated chrome (Cap2 Card / ListRow / Alert / Skeleton)
- [ ] No Cap2 public API changes; no PlanCard / deposit redesign; no Cap3.4 CTA sweep
- [ ] Visual Debt Register updated for any leftovers
- [ ] Pilot Scorecard filled after smoke
- [ ] Lint / tests green

#### Pilot Scorecard (fill during Cap3.3a smoke)

| Check | Status |
|-------|--------|
| Layout | ☐ |
| Contrast | ☐ |
| Responsive | ☐ |
| Keyboard | ☐ |
| CLS | ☐ |
| Scroll | ☐ |

| Smoke (E2E) | Status |
|-------------|--------|
| Navigation (browse plans / TopBar) | ☐ |
| Loading skeleton | ☐ |
| Empty state | ☐ |
| Error state | ☐ |
| List + row navigation | ☐ |
| Balance chip / UserMenu | ☐ |
| Mobile ~390 / narrow Fold | ☐ |

Propagate only when scorecard + smoke are green (**Pilot Freeze** then applies).

#### Visual Debt Register (Cap3.3+)

| Route / area | Debt | Planned |
|--------------|------|---------|
| Cap2 `Card` / `ListRow` | Still hardcode `bg-white` / `border-slate-*` (equals elevated today; not yet bound to `--app-surface-elevated` / `--app-border`) | Cap3.3b or Cap3.5 binding — **no API change** |
| `ListSkeleton` | Duplicates elevated chrome (`rounded-xl`) instead of composing `Card` | Later debt / Cap3.3b if needed |
| `AppPageHeader` | Still uses literal `mb-8` (not spacing SoT alias) | Cap3 close / layout debt |
| Header actions | `DepositCta` secondary sky outline; Cap2 secondary ring-offset not tokenized on all CTAs | Cap3.4 / Cap2 backlog — do not polish in Cap3.3a |
| Other 5 AppShell routes | Page headers still `text-slate-*` / `text-sky-700` on dark shell | **Cap3.3b** (in progress) |
| PlansStore / LocationDetail tabs | `border-slate-200` tab chrome sits on dark shell (not elevated) | Cap3.5 / Tabs Replace — do not invent in Cap3.3b |
| Setup stepper pills | Domain chrome on shell (`bg-sky-700` / slate) | Cap2 Stepper Replace / Cap3.4 — leave |

### Cap3.3b — Propagate

**Done when (pilot):** Cap3.3a scorecard green + Pilot Freeze (**Accepted** 2026-08-05).

Then apply the **same** Golden Route pattern to the remaining five routes (one PR or small sequential follow-ups). Still no CTA theme work (Cap3.4).

**Pattern to copy (only):**

- Page header eyebrow / title / description → `--app-chrome-text` / `--app-chrome-text-muted`
- Shell-adjacent summary chrome (e.g. `CoveragesSummary`) → same tokens
- Elevated Card / ListRow / Alert body text stays slate on light panels
- No TopBar / nav IA change; no landing / auth; no Cap2 API; no cyan CTA sweep

**Out of Cap3.3:** `sky-700` → cyan CTA sweep; Cap2 Review backlog Buttons on setup pages (unless shell chrome only); AuthShell; PlanCard redesign.

### After Cap3 Complete

Write a short **Cap3 Retrospective** (not a new plan): what worked, what Cap4 should do differently, remaining layout debt, lessons for Auth polish / Visual Language. Lives under `docs/design/` next to this plan.

---

## Cap3.4 — CTA theme migration

**Goal:** Store primary CTAs resolve through `--app-primary` (cyan), not hand-rolled sky classes — **without** changing Cap2 public API.

**Allowed:**

- Bind `ui/Button` `tone="app"` **internals** (and `buttonClassName`) to app theme aliases / CSS variables
- Shell-adjacent primary CTAs already on Cap2 Button
- Focus rings that track `--app-focus-ring`

**Not allowed:**

- New Button variants/props
- Landing `.landing-cta*`
- Auth `tone="auth"` redesign (already cyan; leave Cap4)
- Deposit domain outline CTAs / banners (Cap2 intentional exceptions) unless they already use `tone="app"` primary and pick up the token automatically

**Done when:**

- Cap2 `tone="app"` primary renders brand cyan via theme
- No new npm deps
- Spot-check `/plans` + `/me/esims` primary actions

---

## Cap3.5 — Validation + consistency

**Goal:** Close Cap3. No design invention.

### Smoke matrix

| Route | Visual | Responsive | Keyboard | Dark shell | Desktop | Mobile (~390) | Notes |
|-------|--------|------------|----------|------------|---------|---------------|--------|
| `/plans` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | |
| `/[slug]-esim` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | one location |
| `/me/esims` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | loading + empty + list if possible |
| `/me/esims/[id]` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | |
| `/me/esims/[id]/setup` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | |
| `/me/deposit` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | |

### Checklist

- [ ] Dark shell + light elevated surfaces on all six
- [ ] Layout DOM / nav structure unchanged
- [ ] Landing `/` unchanged
- [ ] Auth `/login` `/register` unchanged
- [ ] Cap2 API Freeze held (`git diff` shows no public prop/type API changes on `ui/*` — internal class/token binding OK)
- [ ] `prefers-reduced-motion` still respected where Cap2 motion exists
- [ ] Lint / typecheck / tests green
- [ ] Short Cap3 close note in [status.md](./status.md)

---

## Explicit non-goals (every slice)

- Cap2 migration backlog (raw Buttons/Inputs outside shell CTA path)
- PlanCard, dialogs, popovers, tabs, steppers
- AuthShell / Cap4
- New features, hooks, API calls, nav destinations
- Reopening Design Lock surface model or visual goal

---

## PR hygiene

- Branch from `develop`; PR into `develop`
- Conventional Commits: `feat(ui): Cap3.N …` / `docs(design): …`
- One slice per PR when possible (Cap3.3 may split **pilot** vs **propagate**)
- Squash-merge when CI green

---

## Status

| State | Meaning |
|-------|---------|
| Draft — awaiting acceptance | Superseded |
| **Accepted** | **Current** — Cap3.1 may start |
| Done | Cap3.5 closed Cap3 |

**Next:** Cap3.3a Golden Route (`/me/esims`) → smoke → Pilot Freeze → Cap3.3b propagate. No new design tokens, primitives, or AppShell variants until Cap3 Complete.
