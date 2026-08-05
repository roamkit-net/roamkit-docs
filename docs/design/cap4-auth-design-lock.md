# Cap4 — Auth Polish Design Lock

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-05 |
| Capability | Cap4 — Auth Polish (visual only) |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), Cap2 Review (`roamkit-web/components/ui/CAP2_REVIEW.md`), [Cap3 Design Lock](./cap3-appshell-design-lock.md) (CLOSED), [Implementation Plan](./cap4-implementation-plan.md) |
| Pattern | Same as Landing / Cap3: **Design Lock → Implementation Plan → code** |

This document locks Cap4 decisions so implementation does not wander.  
**Do not reopen locked decisions.** Execution detail lives in the Implementation Plan.

Cap1–Cap3 remain **CLOSED**. Cap4 must not reopen them except for a true production regression.

---

## One-sentence visual goal

> **Auth should feel like the same product as the landing and AppShell while preserving its focused, task-oriented experience.**

Not “copy landing.” Not “merge into AppShell.” Brand alignment with a fast, scannable task UI.

---

## Guardrails (locked)

| Gate | Lock |
|------|------|
| **Auth scope** | `AuthShell` + auth routes only — **not** AppShell, Landing, Billing, Deposit, eSIM |
| **Vizualni cilj** | Same product family; focused task UI (no marketing hero) |
| **Surface model** | Background → Surface → Elevated Surface (**max 3**; no fourth level) |
| **Layout shells** | Landing ≠ Auth ≠ App — three layouts, one Design System |
| **Auth ≠ AppShell** | Auth pages must **not** use `AppShell` |
| **Hierarchy** | Logo → Title → Form → Primary CTA → Secondary links |
| **CTA** | Brand primary via `--auth-*` / Cap2 `tone="auth"` — **no** special auth-only palette |
| **Typography** | `--font-body`; no marketing display / large serif H1 on login |
| **Primitive policy** | Cap2 only (`Button`, `Field`, `Input`, `Alert`, `Card`); no auth-only primitives |
| **Motion** | Existing AppShell/auth motion tokens only; no new auth animation system |
| **Performance** | No new JS dependencies; CSS + existing primitives |
| **Stop rule** | Auth on Brand Design System; layout/flow/validation/API unchanged; Landing + AppShell untouched |

---

## Acceptance criteria (locked)

### 1. Visual polish only

Cap4 may change **only** auth chrome / theme binding:

- AuthShell frame (background, mesh, card surface)
- Title / subtitle / footer link chrome on auth routes
- Cap2 `tone="auth"` internals bound to `--auth-*` (no public API expansion)
- Existing motion that already respects `prefers-reduced-motion`

Cap4 must **not** change auth behaviour, copy, validation rules, Turnstile/Google flows, or redirects.

### 2. Zero feature work

Cap4 must not include:

- new product features
- new API calls or auth endpoints
- new hooks
- new routes or IA
- form validation redesign
- copy rewrites

This is a **pure visual** capability.

---

## What is “auth” (in / out)

### In scope

| Area | Notes |
|------|--------|
| `AuthShell` | Outer frame: dark background, logo, title, subtitle, elevated form card, footer |
| Auth form chrome | Fields, primary submit, error `Alert` — via Cap2 + existing `PasswordField` / Turnstile composition |
| Theme aliases | Restyle / bind `--auth-*` to brand; consume in AuthShell (ADR 016) |

**Auth routes (full surface):**

| Route | Role |
|-------|------|
| `/login` | Sign in |
| `/register` | Create account |
| `/forgot-password` | Request reset |
| `/reset-password` | Complete reset |
| `/set-password` | Set password (invite / first-time) |

### Explicitly out of scope

| Area | Why |
|------|-----|
| Marketing landing (`/`) | Separate layout shell; Cap1/PR2 |
| `AppShell` / store / account routes | Cap3 CLOSED |
| Billing, Deposit, eSIM | Product domains |
| Auth API, tokens, session logic | Behaviour unchanged |
| Form validation rules / error copy | Behaviour / content unchanged |
| Google OAuth / Turnstile product behaviour | Keep; visual chrome only if already in shell |
| Cap2 public API changes | API Freeze — never drive-by |
| New primitives | Cap2 closed |
| Visual debt: tabs, steppers, avatar | **Cap5 Quality Pass** — not Cap4 |
| Iconography / illustrations / hero art | Cap5; Auth forbids marketing hero |
| Light/dark theme toggle | Not Cap4 (ADR 016) |

---

## Three layout shells (locked)

```text
Landing   → marketing homepage (`--landing-*`)
Auth      → focused account tasks (`--auth-*`)
App       → store / account shell (`--app-*`)
```

They share **Brand → Theme → Components → Pages** (ADR 016).  
They do **not** share layout components:

- Auth must not import or wrap with `AppShell`
- Auth must not adopt landing hero / `.landing-cta*` as the primary auth submit path
- App must not adopt `AuthShell`

---

## Auth hierarchy (locked)

Scannable task stack — top to bottom:

```text
Logo
  ↓
Title
  ↓
Form
  ↓
Primary CTA
  ↓
Secondary links
```

**Forbidden on auth pages:**

- Hero media / illustrations
- Marketing sections or promo blocks
- Detached badges / floating callouts
- Display typography as a marketing headline treatment

Login must remain **fast to scan**.

---

## Surface model (locked — max 3)

Same semantic levels as Cap3; auth theme aliases:

```text
Background          (--auth-background)     dark auth canvas
  ↓
Surface             (--auth-surface family) soft frame / mesh context as needed
  ↓
Elevated Surface    light form card          (today: white/95 panel)
```

**Rules:**

- Do **not** invent a fourth token level.
- Do **not** turn the whole viewport into a light elevated page (identity stays dark frame + light card).
- Form card remains Cap2 `Card` **or** existing AuthShell elevated panel bound to `--auth-*` — Implementation Plan chooses bind-vs-migrate without new primitives.
- Contrast: WCAG AA for text on dark frame **and** on elevated form card.

Exact hex / mesh values are fixed in the **Implementation Plan**, not reopened as new goals.

---

## Typography (locked)

| Role | Lock |
|------|------|
| Body / UI | `--font-body` |
| Title | Task title (current scale family); **not** landing display / large serif |
| Display (`--font-display`) | Only if Implementation Plan proves a real need — **default: do not use** on auth |

Auth is not marketing.

---

## CTA (locked)

```text
Brand primary (cyan)
  ↓
--auth-primary / --auth-primary-hover / --auth-primary-foreground
  ↓
Cap2 Button tone="auth"
```

**Forbidden:**

- A special “auth-only” hue outside brand primary
- Hand-rolled `bg-cyan-*` / `bg-sky-*` submit classes that bypass `ui/Button`
- Switching auth submit to `tone="app"` / AppShell chrome
- Expanding Button public API for Cap4

**Allowed:**

- Restyle `--auth-*` values
- Bind `tone="auth"` internals to `--auth-*` without prop changes
- Keep secondary footer links as text links (not a new Button variant)

---

## Layout lock

Baseline today (`AuthShell`):

```text
.auth-page-bg min-h-screen …
  main.auth-shell-enter max-w-md
    Logo → /
    h1 title
    subtitle
    elevated form panel (children)
    footer (secondary links)
```

Cap4 may change **visual tokens** (colors, borders, radius via theme / Cap2, mesh opacity).

Cap4 must **not**:

- Change DOM order of the locked hierarchy
- Widen auth into a marketing or AppShell layout
- Add sidebars, TopBar, or account cluster
- Fork a second AuthShell per route

---

## Primitive & theme policy

1. **Cap2 API Freeze holds.** Cap4 must not edit Cap2 public APIs. Internal theme binding without prop changes is allowed.
2. **Allowed Cap2 primitives on auth:** `Button`, `Field`, `Input`, `Alert`, `Card` (compose; do not invent auth wrappers that duplicate them).
3. **Existing domain fields** (`PasswordField`, `TurnstileField`) stay; Cap4 does not replace their behaviour — only chrome that already flows through Cap2 Input/Field where applicable.
4. **Theme only.** AuthShell chrome consumes `--auth-*`. Brand tokens stay behind theme aliases (ADR 016).
5. **No parallel style systems.** No new `auth-button`, `auth-card` utility families that bypass `ui/*`.

---

## Motion (locked)

- Reuse existing motion tokens (`--motion-*`) and current `.auth-shell-enter` (or equivalent CSS).
- **No new** auth animation library or motion language.
- Must respect `prefers-reduced-motion`.
- Do not gate submit / navigation on animation.

---

## Performance (locked)

- No new npm / client JS dependencies for Cap4.
- Prefer CSS variables + existing Tailwind / Cap2.
- No new image assets or illustration packs.

---

## Proposed implementation slices (outline only)

Not started. Locked as the **preferred PR shape** after Implementation Plan Accepted:

```text
Cap4.1  AuthShell chrome (Golden Route: /login) → Pilot Freeze
Cap4.2  Form theme binding (Field / Input / Button tone=auth; no new props)
Cap4.3  Validation (five auth routes + Landing/App isolation)
```

No Cap4 code until:

1. This Design Lock is **Accepted** ✅
2. Cap4 Implementation Plan is **Accepted**

---

## Stop rule

Cap4 is **CLOSED** when all are true:

1. Auth routes use Brand Design System via `--auth-*` + Cap2 primitives.
2. Locked hierarchy preserved (Logo → Title → Form → Primary CTA → Secondary links).
3. Layout structure unchanged (still AuthShell; **not** AppShell).
4. Auth flow, validation, and API behaviour unchanged.
5. Cap2 primitive **APIs** unchanged (Freeze held).
6. Landing and AppShell **untouched**.
7. Staging smoke on the five auth routes passes (desktop + mobile); Landing + one AppShell route still unchanged.

Anything beyond this is Cap5+ or a separate backlog PR.

---

## Explicitly unchanged (do not reopen in Cap4)

- Cap1 brand token architecture
- Cap2 primitive public APIs and composition rules
- Cap3 AppShell Variant A (CLOSED)
- Landing marketing Design Lock / PR2
- Auth product flows (login/register/password/Google/Turnstile behaviour)
- Visual debt: tabs, steppers, avatar (Cap5)
- Product copy

---

## Entry checklist

Before first Cap4 implementation PR:

- [x] This Design Lock **Accepted**
- [x] Implementation Plan **Accepted** ([cap4-implementation-plan.md](./cap4-implementation-plan.md))
- [ ] Explicit: zero Cap2 API edits in Cap4 PRs
- [ ] Explicit: Landing + AppShell out of diff
- [ ] Explicit: no auth flow / validation / API / copy changes

---

## Status

| State | Meaning |
|-------|---------|
| Draft — awaiting acceptance | Closed |
| **Accepted** | **Current** — do not reopen; Cap4.1 may proceed under Implementation Plan |
| Superseded | Only via explicit redesign decision (new lock) |

**Cap4.1 may start** (Implementation Plan Accepted).
