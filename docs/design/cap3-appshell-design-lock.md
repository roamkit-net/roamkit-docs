# Cap3 — AppShell Variant A Design Lock

| Field | Value |
|-------|-------|
| Status | **Ready for acceptance** (values locked below) |
| Date | 2026-08 |
| Capability | Cap3 — AppShell Variant A (**layout + chrome only**) |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), Cap2 Review |
| Pattern | Design Lock → Implementation Plan → code |

**Do not write Cap3 code until this lock is Accepted.**  
Cap2 remains **CLOSED** (API Freeze). Cap3 must not change Cap2 primitive APIs.

---

## One-sentence visual goal

> **Dark shell + light elevated surfaces.**

Acceptance lens for every Cap3 PR.

---

## Guardrails (locked)

| Gate | Lock |
|------|------|
| Scope | AppShell, TopBar, nav chrome (visual), spacing SoT, `--app-*` surfaces |
| Out | Cap2 APIs, wallet/billing/auth, icons, new nav features, landing |
| Theme | Only `--app-*` in Cap3 chrome; no brand tokens in TSX |
| Cap2 | API Freeze holds |
| Performance | No new JS deps |

---

## 1. Surface palette (locked `--app-*`)

Canonical names follow Cap1 (`--app-background`, not `--app-bg`).

| Token | Locked value | Role |
|-------|--------------|------|
| `--app-background` | `var(--color-background)` → `#05070a` | Page canvas (dark) |
| `--app-shell` | `var(--color-surface)` → `#0d1117` | Shell chrome band (TopBar strip / subtle frame) |
| `--app-surface` | `#ffffff` | Default light panel |
| `--app-surface-elevated` | `#ffffff` | Raised cards (same fill; elevation via shadow) |
| `--app-border` | `#e2e8f0` (slate-200) | Borders on light surfaces / Cap2 Card |
| `--app-border-shell` | `color-mix(in srgb, #fff 10%, transparent)` | Divider on dark shell only |

### Text & action (on surfaces)

| Token | Locked value | Use on |
|-------|--------------|--------|
| `--app-text` | `#0f172a` | Light elevated surfaces |
| `--app-text-muted` | `#475569` | Light elevated surfaces |
| `--app-text-shell` | `#f0f4f8` | Dark background / shell |
| `--app-text-shell-muted` | `#94a3b8` | Dark background / shell |
| `--app-primary` | `var(--color-primary)` → `#22d3ee` | Store CTA (sky → cyan) |
| `--app-primary-hover` | `var(--color-primary-hover)` → `#67e8f9` | |
| `--app-primary-foreground` | `var(--color-primary-foreground)` → `#020617` | |
| `--app-focus-ring` | `var(--focus-ring)` | Focus on both contexts |

**Forbidden:** inventing `--app-page`, `--app-section`, or further color layers beyond this table.

---

## 2. Elevation (locked — two levels max)

| Level | Token | Shadow |
|-------|-------|--------|
| Normal | `--app-surface` | none |
| Elevated | `--app-surface-elevated` | `var(--shadow-sm)` |

No third elevation. No per-page shadow knobs. Cap2 `ui/Card` chrome binds to elevated + `--app-border` in Cap3 wiring (internals only — **no Card API change**).

---

## 3. Width & spacing (SoT in AppShell)

| Token | Locked value | Notes |
|-------|--------------|-------|
| `--app-content-max` | `56rem` (Tailwind `max-w-4xl`) | Default content width |
| `--app-content-max-narrow` | `42rem` (`max-w-2xl`) | Existing `maxWidth="2xl"` |
| `--app-content-max-mid` | `48rem` (`max-w-3xl`) | Existing `maxWidth="3xl"` |
| `--app-gutter-x` | `1.5rem` | Horizontal padding (`px-6` today) |
| `--app-page-padding-y` | `4rem` | Vertical page padding (`py-16` today) |
| `--app-header-gap` | `2rem` | TopBar → page-content (`mt-8` today) |

Pages must not redefine these basics outside `AppShell` / `maxWidth` prop.

---

## 4. TopBar (locked)

| Decision | Lock |
|----------|------|
| Structure | Keep grid `1fr / auto` — `nav` left, `rightSlot` right |
| Height | Content-driven; **no fixed height** in Cap3 |
| Sticky | **No** (not sticky) — revisit only in a later capability |
| Shell / content boundary | `border-b` using `--app-border-shell` under TopBar |
| z-index | `var(--z-header)` if/when sticky is ever added; unused while static |

---

## 5. Contrast audit (locked targets)

| Pair | Requirement |
|------|-------------|
| `--app-text-shell` on `--app-background` / `--app-shell` | WCAG AA ≥ 4.5:1 (body) |
| `--app-text` on `--app-surface-elevated` | WCAG AA ≥ 4.5:1 |
| `--app-primary` + foreground on buttons | AA for UI text / large text as applicable |
| Focus | Visible `focus-visible` ring via `--app-focus-ring` on shell links and elevated controls |
| Hover | Shell nav: lighten muted → shell text; elevated CTAs: `--app-primary-hover` |

Smoke: desktop + mobile on pilot routes with dark shell surrounding white cards — no washed-out borders, no low-contrast muted text.

---

## 6. Motion (locked — one pair)

| Token | Value |
|-------|-------|
| `--app-motion-duration` | `var(--motion-normal)` → `200ms` |
| `--app-motion-ease` | `var(--motion-ease)` → `ease-out` |

If AppShell introduces enter/transition: **only** this duration + easing.  
`prefers-reduced-motion: reduce` → no animation.  
No second duration, no custom curves.

---

## Surface hierarchy (composition)

```text
--app-background          (page)
  └─ --app-shell          (optional chrome band / TopBar strip)
       └─ Page column     (max-width + gutters)
            └─ --app-surface-elevated  (Card)
                 └─ Section            (CardSection)
```

Do not skip (e.g. full-bleed white page as shell).

---

## Explicitly not locked (later)

Icons · illustrations · wallet UI · billing page layouts · dashboard grids · AuthShell (Cap4).

---

## AppShell routes (6)

| Route | Component |
|-------|-----------|
| `/plans` | PlansStore |
| `/[slug]-esim` | LocationDetail |
| `/me/esims` | list |
| `/me/esims/[id]` | detail |
| `/me/esims/[id]/setup` | setup |
| `/me/deposit` | deposit |

Landing + auth = out of Cap3.

---

## Implementation order (after Acceptance)

1. Shell infrastructure (`globals.css` `--app-*` + `AppShell` / `TopBar`)
2. `/plans`
3. `/me/esims`
4. One detail (`/me/esims/[id]` or setup)
5. Smoke (desktop / tablet / mobile / fold)
6. Remaining three AppShell surfaces
7. Cap3 close review

---

## Stop rule

Cap3 CLOSED when:

1. All six AppShell routes use Variant A chrome.
2. Spacing SoT owned by AppShell; no local layout hacks for basics.
3. Cap2 APIs unchanged.
4. No visual regressions outside this planned redesign.
5. Landing + auth untouched.
6. Contrast + reduced-motion checks pass on pilots.

---

## Status

| State | Meaning |
|-------|---------|
| Draft | Superseded by values below |
| **Ready for acceptance** | **Current** — reply **Accepted** / **GO** to flip |
| Accepted | Safe to write Implementation Plan, then Cap3.1 code |
| Superseded | Only via explicit redesign (new lock) |

**Acceptance:** reply Accepted / GO.  
**Next:** short Implementation Plan (wire map), then Cap3.1 shell PR.
