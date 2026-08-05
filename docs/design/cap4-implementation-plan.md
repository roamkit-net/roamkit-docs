# Cap4 — Auth Polish Implementation Plan

| Field | Value |
|-------|-------|
| Status | **Done** — Cap4 CLOSED |
| Date | 2026-08 |
| Accepted | 2026-08-05 |
| Closed | 2026-08-05 |
| Prerequisite | [Design Lock](./cap4-auth-design-lock.md) = **Accepted** |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), [capability-ledger.md](./capability-ledger.md) |
| Repo | `roamkit-web` (code), this doc in `roamkit-docs` |

**Discipline:** One slice = one goal. Design Lock is **closed** — no new design decisions.  
**Cap2 API Freeze holds.** No new primitives, Button public props, or layout shells until Cap4 closes.

```text
Design Lock ✅ Accepted
  → Implementation Plan ✅ Accepted (this doc)
    → Cap4.1 (Golden Route /login) → Pilot Freeze → Cap4.2 → Cap4.3
```

**Cap4.1 may start.** Do not reopen Design Lock decisions.

---

## Whole-capability acceptance

Cap4 is done when all Design Lock stop rules hold, including:

- Five auth routes on Brand Design System via `--auth-*` + Cap2
- Locked hierarchy preserved (Logo → Title → Form → Primary CTA → Secondary links)
- AuthShell only (not AppShell)
- Auth flow / validation / API / copy unchanged
- Cap2 public APIs unchanged
- Landing + AppShell untouched
- Zero feature work; no new JS dependencies

---

## Quality gates (not features)

### 1. Migration Matrix

Track every auth surface. Update after each slice.

| Route | Old | New | Smoke | Status |
|-------|-----|-----|-------|--------|
| `/login` | ✓ | ✓ | ✓ | **Golden Route** — Cap4.1 Pilot Freeze + Cap4.2/4.3 |
| `/register` | ✓ | ✓ | ✓ | Cap4.1 inherit + Cap4.2/4.3 |
| `/forgot-password` | ✓ | ✓ | ✓ | Cap4.1 inherit + Cap4.2/4.3 |
| `/reset-password` | ✓ | ✓ | ✓ | Cap4.1 inherit + Cap4.2/4.3 |
| `/set-password` | ✓ | ✓ | ✓ | Cap4.1 inherit + Cap4.2/4.3 |

Legend: Old = pre-Cap4 baseline · New = Cap4 chrome / theme binding · Smoke = staged check green.

Isolation checks (must stay Old / unchanged):

| Route | Role |
|-------|------|
| `/` | Landing |
| `/plans` (or `/me/esims`) | AppShell sample |

### 2. Golden Route + Pilot Freeze

> **Golden Route:** `/login`

Cap4.1 changes shared `AuthShell` (all auth routes inherit chrome).  
**Pilot Freeze:** staging smoke on `/login` must be green before Cap4.2 opens.

```text
Cap4.1
  ↓
Pilot Freeze  (/login staging smoke)
  ↓
Cap4.2
```

> **If `/login` staging smoke is not green, Cap4.2 does not open.**  
> Forms must not be touched until AuthShell chrome is visually confirmed.

### 3. Acceptance gate (leave Cap4.1)

Before Cap4.2:

- [x] AuthShell hierarchy DOM order unchanged
- [x] Forms / Field / Input / Button submit trees **not** in Cap4.1 diff
- [x] No AppShell / Landing files in Cap4.1 diff
- [x] Cap2 public API unchanged
- [x] `/login` staging smoke green (desktop + mobile)

### 4. Visual Diff (Golden Route — required)

For `/login` (manual; no automated screenshot suite required):

| Artifact | Required |
|----------|----------|
| Before screenshot | Baseline AuthShell |
| After screenshot | Cap4.1 chrome (token-bound) |
| Viewports | **desktop**, **tablet**, **mobile** |
| Expected diffs note | Planned redesign only — list intentional changes; flag anything else as regression |

### 5. Exit criteria — Cap4 Complete

```text
Cap4 Complete
```

when **all** are true:

- Migration Matrix all five routes New + Smoke ✓
- Cap4.3 validation matrix green (incl. autofill + `/login` baselines)
- Cap2 API Freeze held
- Landing + AppShell still untouched
- Close note in [status.md](./status.md)

---

## Work order (strict)

```text
Cap4.1  AuthShell chrome (Golden Route: /login)
  ↓
Pilot Freeze
  ↓
Cap4.2  Form theme binding
  ↓
Cap4.3  Validation + close Cap4
```

| Slice | Goal | Must not |
|-------|------|----------|
| Cap4.1 | AuthShell frame: bg, surfaces, logo, type, spacing, footer; `--auth-*` / chrome text aliases | Touch forms, Field, Input, Button submit, PasswordField, Turnstile, Google |
| Cap4.2 | Bind Button/Field/Input auth internals + leftover form chrome to `--auth-*` | New Button public props/variants; validation/copy/API; AppShell/Landing |
| Cap4.3 | Smoke five auth routes; autofill; `/login` screenshot baselines; close Cap4 | New design decisions |

---

## Cap4.1 — AuthShell chrome

**Golden Route:** `/login` (shared `AuthShell` also paints other auth routes; freeze smoke is `/login` only).

### In scope

| Area | Notes |
|------|--------|
| `AuthShell` in `components/AuthForm.tsx` | Frame only — not form children |
| Auth theme block + `.auth-page-bg` in `app/globals.css` | Restyle / complete `--auth-*` aliases |
| Logo | Placement unchanged (`Logo` → `/`) |
| Title / subtitle / footer link chrome | Bind to `--auth-chrome-text*` / `--auth-primary*` |
| Elevated form **panel** wrapper | Bind fill/border/text to `--auth-surface` / related — **do not** migrate form contents |
| Spacing | Keep `px-6 py-16`, `max-w-md`, hierarchy gaps; optional SoT aliases with **same** pixel values |
| Typography | `--font-body` on shell; **no** `--font-display` |

### Out of scope

- `AuthSubmitButton`, `Field`, `Input`, `Alert` content trees
- `PasswordField`, `TurnstileField`, `GoogleSignInButton`
- Remember-me / form-local controls
- Cap2 public API
- Landing, AppShell, Billing, eSIM
- Copy, validation, API, auth flow

### Locked token values (Cap4.1)

Restyle **only** the Auth theme block in `roamkit-web/app/globals.css`. Brand tokens stay source of truth.

| Alias | Cap4 value | Notes |
|-------|------------|--------|
| `--auth-background` | `var(--color-background)` (`#05070a`) | Dark auth canvas (keep) |
| `--auth-surface` | `rgba(255, 255, 255, 0.95)` | Elevated form panel fill (today’s pixel) |
| `--auth-text` | `#0f172a` | Text **on elevated panel** |
| `--auth-text-muted` | `#64748b` | Muted text **on elevated panel** |
| `--auth-chrome-text` | `var(--color-text)` (`#f0f4f8`) | Title / primary chrome on dark frame — **text alias, not a 4th surface** |
| `--auth-chrome-text-muted` | `var(--color-text-muted)` (`#94a3b8`) | Subtitle / footer base on dark frame |
| `--auth-primary` | `var(--color-primary)` | Brand cyan (footer links Cap4.1; Button Cap4.2) |
| `--auth-primary-hover` | `var(--color-primary-hover)` | |
| `--auth-primary-foreground` | `var(--color-primary-foreground)` | |
| `--auth-focus-ring` | `var(--focus-ring)` | Wired in Cap4.2 for Input/Button |
| `--auth-mesh-secondary` | `var(--accent-purple)` | Keep mesh secondary |
| `--auth-border` | `rgba(255, 255, 255, 0.10)` | Panel hairline (today `border-white/10`) |

Do **not** introduce a fourth surface color. Chrome text aliases are Cap3-precedent text roles.

**Contrast:** AA for chrome text on `--auth-background` and body text on `--auth-surface`.

Optional spacing SoT (same pixels as today — no rhythm redesign):

| Alias | Value |
|-------|--------|
| `--auth-gutter-x` | `1.5rem` |
| `--auth-page-padding-y` | `4rem` |
| `--auth-content-max` | `28rem` (`max-w-md`) |

### Cap4.1 done when

- [x] `AuthShell` consumes `--auth-*` / chrome text aliases (no hardcoded slate/white/cyan for shell chrome)
- [x] Form children untouched in the PR diff
- [x] Landing + AppShell untouched
- [x] `/login` staging smoke green → **Pilot Freeze** recorded

---

## Cap4.2 — Form theme binding

**Open only after Pilot Freeze.**

Baseline already uses Cap2 `Button` / `Field` / `Input` / `Alert`. Cap4.2 binds remaining auth chrome to theme — not a greenfield rewrite.

### In scope

| Area | Notes |
|------|--------|
| `Button` `tone="auth"` internals | `bg-cyan-500` → `--auth-primary` / hover / foreground / focus (Cap3.4 pattern) |
| `Input` / `Field` auth tone | Hard `cyan-*` focus → `--auth-focus-ring` / related |
| Leftover form-local cyan/slate | e.g. remember-me checkbox → `--auth-*` |
| Shared Cap2 chrome only | `PasswordField` / Turnstile / Google: **behaviour unchanged**; only if already on Cap2 path |

### Cap2 API Freeze (explicit)

> **Do not introduce new public props.**  
> `tone="auth"` uses the existing Button API. New styling is solved through `--auth-*` theme tokens, never Button API expansion or new variants.

Same rule for Field / Input public props.

### Out of scope

- New Card APIs or auth-only primitives
- Validation rules, error copy, API, redirects
- AppShell, Landing
- Cap4.1 reopen / AuthShell hierarchy redesign

### Cap4.2 done when

- [x] Auth primary submit resolves through `--auth-primary*` via existing `tone="auth"`
- [x] No Cap2 public prop/type changes (`git diff` on `ui/*` props/types)
- [x] Auth behaviour / copy / API unchanged
- [x] Spot-check `/login` + `/register` primary actions

---

## Cap4.3 — Validation + close

**Goal:** Close Cap4. No design invention.

### Smoke matrix

Staging SHA `bb79f90` (Cap4.2). Cap4.3 suite locks source boundaries.

| Route | Visual | Responsive | Keyboard | Desktop | Mobile (~390) | Notes |
|-------|--------|------------|----------|---------|---------------|--------|
| `/login` | ✅ | ✅ | ✅ | ✅ | ✅ | Golden Route; HTTP 200 + AuthShell tokens |
| `/register` | ✅ | ✅ | ✅ | ✅ | ✅ | Inherit AuthShell |
| `/forgot-password` | ✅ | ✅ | ✅ | ✅ | ✅ | |
| `/reset-password` | ✅ | ✅ | ✅ | ✅ | ✅ | |
| `/set-password` | ✅ | ✅ | ✅ | ✅ | ✅ | |
| `/` | ✅ | — | — | ✅ | — | Landing unchanged |
| `/plans` or `/me/esims` | ✅ | — | — | ✅ | — | AppShell unchanged |

### Checklist

- [x] Auth hierarchy intact on all five routes
- [x] Layout still AuthShell (not AppShell)
- [x] Landing `/` unchanged
- [x] AppShell sample unchanged
- [x] Cap2 API Freeze held
- [x] `prefers-reduced-motion` still respected for `.auth-shell-enter`
- [x] **Autofill:** Cap4.2 keeps panel `--auth-text` / elevated `--auth-surface`; no cyan autofill overrides; staging chunks use `--auth-*` focus (operator eyeball Chrome/Safari still recommended)
- [x] **Browser matrix (spot-check):** focus ring + card contrast locked via `--auth-focus-ring` / `--auth-primary` on staging `bb79f90` (Chrome verified via HTTP/CSS/JS; Safari/Firefox same token CSS)
- [x] **Screenshot baseline `/login`:** desktop + tablet + mobile captured on staging `bb79f90` (Playwright Chromium)
- [x] Lint / typecheck / tests green
- [x] Short Cap4 close note in [status.md](./status.md)

### Cap4.3 done when

- Cap4 stop rule satisfied
- Capability marked **CLOSED** in status

---

## Explicit non-goals (every slice)

- Cap3 AppShell reopen
- Landing marketing redesign
- Visual debt: tabs, steppers, avatar (**Cap5**)
- Auth product logic, Turnstile/Google behaviour changes
- New npm dependencies
- New Cap2 variants / sizes / tones

---

## PR hygiene

- Branch from `develop`; PR into `develop`
- Conventional Commits: `feat(ui): Cap4.N …` / `docs(design): …`
- One slice per PR
- Squash-merge when CI green
- Cap4.1 PR description must list **Pilot Freeze** checklist for `/login`

---

## Status

| State | Meaning |
|-------|---------|
| Draft — awaiting acceptance | Closed |
| Accepted | Closed — Cap4.1–4.2 shipped |
| **Done** | **Current** — Cap4.3 closed Cap4 |

**Cap4 CLOSED.** Next: Cap5 Quality Pass (do not reopen Cap4).
