# Cap5 — Design Consistency Polish Design Lock

| Field | Value |
|-------|-------|
| Status | **Draft — awaiting acceptance** |
| Date | 2026-08 |
| Capability | Cap5 — Design Consistency Polish (visual chrome only) |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), Cap2 Review (`roamkit-web/components/ui/CAP2_REVIEW.md`), [Cap3 Design Lock](./cap3-appshell-design-lock.md) (CLOSED), [Cap4 Design Lock](./cap4-auth-design-lock.md) (CLOSED) |
| Pattern | Same as Cap1–Cap4: **Design Lock → Implementation Plan → code** |
| Prerequisite | Design System **v1.0** (Cap1–Cap4) CLOSED |

This document locks Cap5 decisions so implementation does not wander.  
**Do not reopen locked decisions.** Execution detail lives in the Implementation Plan (not yet drafted).

Cap1–Cap4 remain **CLOSED**. Cap5 must not reopen them except for a true production regression.

---

## One-sentence visual goal

> **Eliminate remaining legacy UI chrome so every authenticated screen follows the same visual language.**

Emphasis: **consistency**, not redesign.

---

## Context (locked)

```text
Brand Design System v1.0 ✅

Cap1  Tokens                ✅
Cap2  Shared Primitives     ✅
Cap3  AppShell Variant A    ✅
Cap4  Auth Polish           ✅

FOUNDATION + APP + AUTH COMPLETE
```

Cap5 does **not** build the Design System. It finishes visual consistency of leftover chrome identified during Cap2/Cap3 review.

After Cap5 → Cap6 Cross-Route Consistency Review (audit only).

---

## Guardrails (locked)

| Gate | Lock |
|------|-------|
| **Scope** | Tabs, stepper pills, avatar / profile chrome, Cap2/Cap3 backlog legacy chrome only |
| **Vizualni cilj** | Same visual language on authenticated screens; no redesign |
| **Primitives** | Existing Cap2 only — **no new `ui/*` primitives** |
| **Tokens** | Existing `--app-*` / `--auth-*` / `--landing-*` / brand only — **no new tokens** |
| **Layouts** | Chrome only — **no** DOM hierarchy / structure / IA changes |
| **Tabs** | Same API, same navigation, same state — theme only |
| **Stepper** | Active / completed / inactive chrome only — no new logic |
| **Avatar** | Surface, border, radius, focus only — **not** dropdown behaviour |
| **Cap2 API Freeze** | Hold — no new public props, variants, sizes, or tones |
| **Performance** | No new JS dependencies |
| **Stop rule** | Backlog chrome cleared; no new tokens/primitives/layouts/features |

---

## Acceptance criteria (locked)

### 1. Consistency polish only

Cap5 may change **only**:

- Tab indicator / idle / active chrome → `--app-*` (or existing Cap2 composition)
- Setup stepper pill chrome (active / completed / inactive) → `--app-*`
- UserMenu avatar trigger chrome (fill, border, radius, focus ring / offset)
- Other Cap2/Cap3 **registered** legacy chrome listed in the backlog below

Cap5 must **not** change behaviour, routing, copy, API, or product UX.

### 2. Zero feature work

Cap5 must not include:

- new product features
- new routes or IA
- new hooks or API calls
- copy rewrites
- dialog / popover / menu redesign
- PlanCard or Deposit UX redesign

### 3. No Design System expansion

Forbidden:

- new `components/ui/*` primitives (no `ui/Tabs`, `ui/Stepper`, `ui/Avatar` in Cap5)
- new CSS custom properties / theme aliases beyond existing Cap1–Cap4 sets
- new Cap2 Button/Input tones or variants

Theme binding uses existing `--app-*` (authenticated AppShell surfaces). Do not invent Cap5-only palettes.

---

## What is in / out

### In scope (Visual Debt backlog)

| Element | Location (today) | Cap5 change |
|---------|------------------|-------------|
| **Tabs** | `PlansStore` tablist (`bg-sky-700` active); related tab chrome on AppShell catalog surfaces if same pattern | Theme binding only; keep `role="tablist"` / URL `tab` param / labels |
| **Stepper pills** | `/me/esims/[id]/setup` step pills (`bg-sky-700` / `bg-sky-100` / `bg-slate-200`) | Active / completed / inactive → `--app-*` (or brand via `--app-*`); no step machine changes |
| **Avatar / profile chrome** | `UserMenu` circular trigger (`bg-sky-700`, `ring-sky-500`) | Surface, border, radius, focus ring + `--app-background` offset |
| **Registered Cap3 allowlist** | Cap3.5 `SKY700_ALLOWLIST` rows for the three above | Clear allowlist when chrome is token-bound |

Cap3.5 currently allowlists:

- `app/me/esims/[id]/setup/page.tsx` (stepper)
- `components/PlansStore.tsx` (tabs)
- `components/UserMenu.tsx` (avatar)

Those three are the **primary Cap5 backlog**. Additional “legacy chrome” only if it was already named in Cap2 inventory / Cap3 Visual Debt Register as **tabs / stepper / avatar** — not a free sweep of every `sky-*` in the repo.

### Explicitly out of scope

| Area | Why |
|------|-----|
| Landing (`/`) | Cap1 / marketing shell CLOSED |
| Auth routes / `AuthShell` | Cap4 CLOSED |
| `AppShell` layout / TopBar / nav IA | Cap3 CLOSED |
| PlanCard redesign | Product card — Cap6+ or separate |
| Deposit UX / banners / EIP-681 panels | Product domain |
| Dialog redesign | Out |
| Popover / dropdown menu redesign | Avatar **menu panel** stays; only trigger chrome in Cap5 |
| `text-sky-700` text links / eyebrows (DepositCta, PlanCard, explorers, etc.) | Cap6 audit / follow-ups — not Cap5 invent |
| API, routing, copy | Behaviour / content unchanged |
| New primitives (`ui/Tabs`, etc.) | Cap2 closed; Cap5 binds in place |
| Light/dark theme toggle | Not Cap5 (ADR 016) |
| Cap6 full-repo color/spacing/radius/shadow audit | After Cap5 |

---

## Locked decisions (per element)

### Tabs

```text
Same API
Same navigation
Same state
→ Theme only
```

- Keep existing tab IDs, labels, `searchParams` behaviour, and a11y roles.
- Active / idle / hover chrome binds to `--app-*` (primary / surface / chrome text as appropriate on dark shell vs elevated).
- Do **not** extract a shared `ui/Tabs` primitive in Cap5.
- Do **not** change tab count, order, or filter semantics.

### Stepper

```text
active | completed | inactive
→ Chrome only
```

- Three visual states only; no new states, animations systems, or step logic.
- Bind fills/text to existing `--app-*` / brand-via-app aliases.
- Do **not** change step count, order, or setup flow content.

### Avatar

```text
surface | border | radius | focus
→ Trigger chrome only
```

- Token-bind the circular trigger (and focus ring / ring-offset).
- Do **not** change open/close, menu items, logout, or panel layout/styles beyond what is required so the trigger is consistent (prefer leave menu panel alone).
- Do **not** introduce an Avatar primitive or image-upload UI.

---

## Theme policy (locked)

| Surface | Tokens |
|---------|--------|
| AppShell authenticated chrome | `--app-*` |
| Auth | `--auth-*` — **do not reopen** |
| Landing | `--landing-*` — **do not reopen** |
| Brand source | Cap1 brand tokens — consume **via theme aliases**, not scatter brand literals in Cap5 call sites |

Same Cap3 rule: prefer theme aliases in components over raw `--color-*` at call sites when binding chrome.

---

## Cap2 API Freeze (locked)

Cap5 holds the freeze:

- No new public props on `ui/*`
- No new variants / sizes / tones
- No business/domain props on primitives
- In-place class/token binding on existing call sites only

---

## Chrome consistency matrix (validation gate)

Cap5 closes when this matrix is green for **authenticated App** chrome (not Landing/Auth redesign):

| Element | Result |
|---------|--------|
| Button | ✓ (Cap2/Cap3 — must stay green) |
| Input | ✓ (Cap2 — must stay green) |
| Alert | ✓ (Cap2 — must stay green) |
| Card | ✓ (Cap2/Cap3 — must stay green) |
| Tabs | ✓ Cap5 |
| Stepper | ✓ Cap5 |
| Avatar | ✓ Cap5 |

Regression: Landing `/` and Auth routes remain unchanged.

---

## Stop rule

Cap5 is done when **all** are true:

```text
Legacy Cap5 backlog chrome cleared     ✅
No new tokens                          ✅
No new primitives                      ✅
No layout / structure changes          ✅
No feature work                        ✅
Chrome consistency matrix green        ✅
Cap2 API Freeze held                   ✅
Landing + Auth untouched               ✅
```

Then mark Cap5 **CLOSED** in [status.md](./status.md). Cap6 may open (audit only).

---

## Explicit non-goals

- Reopening Cap1–Cap4
- “Cleanup” PRs outside the backlog table
- Full `sky-*` repo purge (that is Cap6 audit → follow-ups)
- Marketing / illustration / iconography system
- New motion systems

---

## Process (locked)

```text
Design Lock (this doc) — Draft → Accepted
  → Implementation Plan — Draft → Accepted
    → Cap5.N slices (one concern per PR)
      → Validation + close
```

Do **not** open `roamkit-web` Cap5 code until Design Lock is **Accepted** and Implementation Plan is **Accepted**.

Suggested slice shape (non-binding until Plan):

```text
Cap5.1  Tabs theme
Cap5.2  Stepper pills theme
Cap5.3  Avatar trigger chrome
Cap5.4  Validation + close (matrix + Cap3.5 allowlist cleared)
```

Exact slices live in the Implementation Plan.

---

## Status

| State | Meaning |
|-------|---------|
| **Draft — awaiting acceptance** | **Current** |
| Accepted | Cap5 Implementation Plan may be drafted |
| Done | Cap5.N closed Cap5 |

**Next after Accept:** Cap5 Implementation Plan draft (still no web code until Plan Accepted).
