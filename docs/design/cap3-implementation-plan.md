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
| `/me/esims` | ✓ | ⏳ | ⏳ | **pilot** (Cap3.3a) |
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

**Cap3.1 wiring:** `AppShell` outer frame uses `--app-background` + chrome text tokens (via CSS vars / theme classes). Do **not** migrate cards or Button in Cap3.1.

Baseline to replace in `AppShell.tsx`:

```text
min-h-screen bg-slate-50 px-6 py-16 text-slate-900
```

Spacing SoT (`px-6`, `py-16`, `mt-8`, `maxWidth`) stays unless a tiny rhythm tweak is required for contrast — **no DOM hierarchy change**.

---

## Cap3.1 — Shell tokens + background

**Files (expected):** `app/globals.css`, `components/AppShell.tsx`, `AppShell` tests if class assertions exist.

**Done when:**

- `--app-*` values match the table above (plus chrome text aliases)
- All AppShell routes show dark shell background (cards may still look “old light-on-light” until Cap3.3 — acceptable interim)
- Landing + auth CSS/pages unchanged
- Lint/tests green

**Out:** TopBar retint beyond inheriting shell text; Card migration; Button CTA.

---

## Cap3.2 — TopBar + navigation chrome

**Files (expected):** `components/TopBar.tsx`, callers’ `nav` link classes only if required for contrast on dark shell, `AuthNav` **app** presentation if it sits in `rightSlot` (not landing variant).

**Done when:**

- TopBar / nav / account chrome readable on dark shell (`--app-chrome-text*`)
- Grid contract unchanged (`nav` left, `rightSlot` right)
- No new nav items or IA

**Out:** Page body cards; PlanCard; deposit panels.

---

## Cap3.3 — Surface migration

**Goal:** Light elevated surfaces on dark shell using Cap2 `ui/Card` / existing elevated chrome — **no Cap2 API changes**.

### Pilot rule (acceptance — required)

> **One route is the pilot.** After the pilot is accepted and staging smoke passes, apply the same pattern to the remaining AppShell routes.

Do **not** propagate Variant A surfaces to all six routes in the same step as the first visual call.

| Step | Route | Gate |
|------|-------|------|
| Pilot | **`/me/esims`** | Visual accept + staging smoke |
| Propagate | `/plans`, `/[slug]-esim`, `/me/esims/[id]`, `/me/esims/[id]/setup`, `/me/deposit` | Same surface pattern only |

**Why `/me/esims`:** Account shell with Card / Skeleton / Empty — exercises elevated surfaces without PlanCard catalog density.  
Prefer **two PRs** inside Cap3.3: `Cap3.3a` pilot, `Cap3.3b` propagate (after pilot GO).

**Done when (pilot):**

- Pilot route: content sits on `--app-surface-elevated` (via `ui/Card` / shell-owned wrappers)
- No PlanCard redesign; no deposit banner redesign
- Staging smoke on pilot green → **then** propagate PR(s)

**Propagate may be one PR or small follow-ups** — still Cap3.3 scope, still no CTA theme work (that is Cap3.4).

**Out:** `sky-700` → cyan CTA sweep; Cap2 Review backlog Buttons on setup pages (unless they are shell chrome only).

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

| Route | Desktop | Mobile (~390) | Notes |
|-------|---------|---------------|--------|
| `/plans` | ☐ | ☐ | |
| `/[slug]-esim` | ☐ | ☐ | one location |
| `/me/esims` | ☐ | ☐ | loading + empty + list if possible |
| `/me/esims/[id]` | ☐ | ☐ | |
| `/me/esims/[id]/setup` | ☐ | ☐ | |
| `/me/deposit` | ☐ | ☐ | |

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

**Next:** Cap3.1 only — shell tokens + background. No new design tokens, primitives, or AppShell variants until Cap3 Complete.
