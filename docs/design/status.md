# Design System status

RoamKit web primitives (ADR 016).

**Cap2 CLOSED** — quality gate: [`roamkit-web/components/ui/CAP2_REVIEW.md`](https://github.com/roamkit-net/roamkit-web/blob/develop/components/ui/CAP2_REVIEW.md)  
Colocated mirror: `roamkit-web/components/ui/README.md`.

| Primitive | Status | Canonical | PR |
|-----------|--------|-----------|-----|
| Button | ✅ | `ui/Button` | [#97](https://github.com/roamkit-net/roamkit-web/pull/97) |
| Alert | ✅ | `ui/Alert` | [#98](https://github.com/roamkit-net/roamkit-web/pull/98) |
| Field | ✅ | `ui/Field` | Cap2.3 — [#99](https://github.com/roamkit-net/roamkit-web/pull/99) |
| Input | ✅ | `ui/Input` | [#99](https://github.com/roamkit-net/roamkit-web/pull/99) |
| Textarea | ✅ | `ui/Textarea` | Cap2.3 |
| Card | ✅ | `ui/Card` (+ Header / Section / Footer) | [#100](https://github.com/roamkit-net/roamkit-web/pull/100) |
| ListRow | ✅ | `ui/ListRow` | [#100](https://github.com/roamkit-net/roamkit-web/pull/100) |
| Skeleton | ✅ | `ui/Skeleton` | [#101](https://github.com/roamkit-net/roamkit-web/pull/101) |
| Empty | ✅ | `ui/Empty` | [#101](https://github.com/roamkit-net/roamkit-web/pull/101) |
| Badge | ✅ | `ui/Badge` | [#101](https://github.com/roamkit-net/roamkit-web/pull/101) |

## Cap2 API Freeze

- New props only with justification.
- Breaking changes only via a new capability / ADR — **never drive-by** in later caps.
- No business / domain props on `ui/*`.
- Cap3–Cap4 held the freeze (internal class/token binding only).

## Rules

- Brand → Theme → Components → Pages ([ADR 016](../adr/016-web-design-tokens.md))
- Migration: create primitive → tests → migrate callers → delete duplicates
- Cap2 was architecture-only — no intentional visual change
- Alert ≠ Toast
- Card is a **container** only (no billing/auth/pricing variants; no elevation API)
- Composition: `Button → Field → Card → Page`

## Cap3 — AppShell Variant A — **CLOSED**

| Doc | Status |
|-----|--------|
| [Design Lock](./cap3-appshell-design-lock.md) | **Accepted** |
| [Implementation Plan](./cap3-implementation-plan.md) | **Done** |

| Slice | Status | PR |
|-------|--------|-----|
| Cap3.1 Shell tokens + background | ✅ | [web #103](https://github.com/roamkit-net/roamkit-web/pull/103) |
| Cap3.2 TopBar + nav chrome | ✅ | [web #104](https://github.com/roamkit-net/roamkit-web/pull/104) |
| Cap3.3a Pilot `/me/esims` | ✅ Pilot Freeze | [web #105](https://github.com/roamkit-net/roamkit-web/pull/105) |
| Cap3.3b Propagate | ✅ staging smoke `eaf4c0c` | [web #110](https://github.com/roamkit-net/roamkit-web/pull/110) |
| **Cap3.3 (surfaces)** | **CLOSED** | 3.3a + 3.3b |
| Cap3.4 CTA theme | ✅ | [web #111](https://github.com/roamkit-net/roamkit-web/pull/111) |
| Cap3.5 Validation | ✅ | [web #112](https://github.com/roamkit-net/roamkit-web/pull/112) |
| **Cap3 (AppShell)** | **CLOSED** | Cap3.1–3.5 |

### Cap3 close note

| Stop-rule criterion | Result |
|---------------------|--------|
| Dark shell + light elevated surfaces | ✅ Cap3.3 |
| Brand primary CTAs via `--app-primary` | ✅ Cap3.4 — staging `6333899` (`bg-[var(--app-primary)]` in client chunks) |
| Cap2 primitive APIs unchanged | ✅ Freeze held |
| Landing `/` unchanged | ✅ `landing-cta` / `--landing-ink` |
| Auth `/login` unchanged | ✅ `auth-page-bg` / `tone="auth"` |
| Validation | ✅ Cap3.5 suite + cross-route HTTP 200 |

CTA contrast (`#22d3ee` on `#020617`): **11.16:1**.

**Visual Debt (not Cap3):** PlansStore tab indicator, setup stepper pills, UserMenu avatar — still `sky-700`.

Cap2 Merge leftovers remain migration backlog.

## Cap4 — Auth Polish — **CLOSED**

| Doc | Status |
|-----|--------|
| [Design Lock](./cap4-auth-design-lock.md) | **Accepted** |
| [Implementation Plan](./cap4-implementation-plan.md) | **Done** |
| [Capability Ledger](./capability-ledger.md) | Cap history (not current status) |

| Slice | Status | PR |
|-------|--------|-----|
| Cap4.1 AuthShell chrome | ✅ Pilot Freeze | [web #113](https://github.com/roamkit-net/roamkit-web/pull/113) |
| Cap4.2 Form theme binding | ✅ staging `bb79f90` | [web #114](https://github.com/roamkit-net/roamkit-web/pull/114) |
| Cap4.3 Validation | ✅ | Cap4.3 suite (this close) |
| **Cap4 (Auth Polish)** | **CLOSED** | Cap4.1–4.3 |

### Cap4 close note

| Stop-rule criterion | Result |
|---------------------|--------|
| AuthShell via `--auth-*` | ✅ Cap4.1 — staging Pilot Freeze on `/login` (`41bd98e`) |
| Form primary / focus / remember-me via `--auth-*` | ✅ Cap4.2 — staging `bb79f90` (`var(--auth-primary)` in client chunks; no `bg-cyan-500`) |
| Canonical Cap2 primitives (`tone="auth"`) | ✅ No new public props |
| Landing `/` unchanged | ✅ `--landing-ink` / `landing-cta`; no `auth-page-bg` |
| AppShell `/plans` unchanged | ✅ `--app-chrome-text`; no AuthShell |
| Validation | ✅ Cap4.3 suite + five auth routes HTTP 200 on staging `bb79f90` |

```text
AuthShell             ✅
Auth theme            ✅
Canonical primitives  ✅
Landing unchanged     ✅
AppShell unchanged    ✅
Validation passed     ✅
```

Brand primary CTA contrast (`#22d3ee` on dark auth chrome): **11.16:1** (same brand primary as Cap3).

**Visual Debt (Cap5):** PlansStore tabs, setup stepper, UserMenu avatar — out of Cap4 scope.

## Next

**Cap5 — Quality Pass** (tabs / stepper / avatar Visual Debt, then Cap6 Consistency Review).

Brand Design System: Cap1–Cap4 closed. Cap5 may open.

## Inventory

[component-inventory.md](./component-inventory.md) (Cap2b — historical)  
[capability-ledger.md](./capability-ledger.md) (Cap1–CapN PR index)
