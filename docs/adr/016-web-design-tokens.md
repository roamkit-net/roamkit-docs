# ADR 016: RoamKit web design tokens

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-08 |
| Deciders | Brand Design System Cap1 (post-Landing PR2) |
| Repos | `roamkit-web` (`app/globals.css`), this ADR |

## Context

Landing PR2 introduced scoped `--landing-*` tokens and marketing utilities. AppShell (store/account) and AuthShell still use ad-hoc Tailwind slate/sky/cyan. Copying landing tokens into AppShell would create three parallel palettes (landing / app / auth).

We need one brand layer and theme aliases before any AppShell visual refresh (Cap3 Variant A: dark shell + light cards).

## Decision

### Architecture

```text
Brand tokens
  → Theme aliases (landing | app | auth)
    → Components
      → Pages
```

**Token ownership:** Brand tokens are the only source of truth. Theme aliases may reference brand tokens; component and page styles must never redefine brand colors.

**Forbidden:** Components and pages must not reference brand tokens directly. Always consume theme aliases (or Cap2+ primitives bound to a theme).

### Naming

Semantic only (`--color-primary`, `--color-surface`). Never hue names (`--cyan`, `--ink`, `--slate`). Brand may change the underlying hue without renaming tokens.

Future light/dark switch must not require renames (`--color-surface` good; `--light-surface` forbidden). Cap1 does **not** implement a theme toggle.

### Brand inventory (Cap1)

Declared in `roamkit-web/app/globals.css`:

- Color: `--color-background`, `--color-surface`, `--color-surface-elevated`, `--color-text`, `--color-text-muted`, `--color-primary`, `--color-primary-hover`, `--color-primary-foreground`, `--focus-ring`
- Motion: `--motion-fast`, `--motion-normal`, `--motion-slow`, `--motion-ease`
- Radius: `--radius-sm`, `--radius-md`, `--radius-lg` (Cap2+: Card → lg; Button/Input → md)
- Shadow: `--shadow-sm`, `--shadow-md`, `--shadow-lg`
- Z-index: `--z-header`, `--z-dropdown`, `--z-modal`, `--z-toast`
- Typography: `--font-body`, `--font-display`
- Icon scale: `--icon-size-sm|md|lg|xl` (16 / 20 / 24 / 32) — assets later (Cap5)

Brand primary = landing cyan (`#22d3ee` / hover `#67e8f9`).

### Themes

| Theme | Role | Cap1 notes |
|-------|------|------------|
| Landing | Marketing homepage | `--landing-*` aliases brand; utilities `.landing-*` unchanged |
| App | Store / account (`AppShell`) | `--app-*` mirrors today’s light slate-50 / sky-700; unused by TSX until Cap2/Cap3 |
| Auth | Login / register / password flows | `--auth-*`; `.auth-page-bg` uses theme vars |

App primary may temporarily differ (sky-700) until Cap3 unifies store primary to brand cyan.

### Deprecation policy

```text
Legacy tokens
  → @deprecated (keep working)
    → No new usages
      → Migration (Cap2–Cap4)
        → Removal (after Cap3/Cap4 — not Cap1)
```

`--landing-*` remains a **legacy public API** for existing marketing components. New components must not add fresh `--landing-*` or hardcoded brand hex. Remove legacy names only after Cap3/Cap4 migration.

### Cap1 stop rule

Cap1 is architectural only:

- Zero intentional pixel change
- Manual before/after smoke on `/`, `/plans`, `/login`, `/register`, `/me/*`
- Diff = CSS architecture + this ADR only

### Roadmap (separate PRs)

1. Cap1 — tokens + this ADR (this decision) — **done**
2. Cap2b — component inventory with Status: Reuse / Merge / Replace / Delete (before Cap2)
3. Cap2 — shared UI primitives (theme-bound, not brand-direct; zero visual)
4. Cap3 — AppShell Variant A (dark shell + light elevated cards)
5. Cap4 — Auth polish onto shared tokens/primitives
6. Cap5 — iconography / empty states / illustrations
7. Cap6 — consistency review

Inventory live doc: [component-inventory.md](../design/component-inventory.md).

## Consequences

### Positive

- One brand source of truth; themes cannot drift into parallel palettes
- AppShell restyle (Cap3) can swap theme values without inventing a fourth system
- Governance + deprecation path prevents half-migrated dual APIs

### Negative / follow-ups

- Until Cap2/Cap3, product TSX still hardcodes slate/sky classes (expected)
- Sky vs cyan primary split remains until Cap3
- `@theme inline` color utilities use literals equal to brand to avoid a CSS variable cycle with legacy `--background`

## References

- `roamkit-web/app/globals.css`
- Landing PR2 (scoped `--landing-*`, Design Lock)
- AppShell PR (`components/AppShell.tsx`)
