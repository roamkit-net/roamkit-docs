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
- Breaking changes only via a new capability / ADR — **never drive-by in Cap3**.
- No business / domain props on `ui/*`.
- Cap3 must not change Cap2 primitive APIs.

## Rules

- Brand → Theme → Components → Pages ([ADR 016](../adr/016-web-design-tokens.md))
- Migration: create primitive → tests → migrate callers → delete duplicates
- Cap2 was architecture-only — no intentional visual change
- Alert ≠ Toast
- Card is a **container** only (no billing/auth/pricing variants; no elevation API)
- Composition: `Button → Field → Card → Page`

## Next

**Cap3 — AppShell Variant A**

| Doc | Status |
|-----|--------|
| [Design Lock](./cap3-appshell-design-lock.md) | **Accepted** |
| [Implementation Plan](./cap3-implementation-plan.md) | **Accepted** |

| Slice | Status | PR |
|-------|--------|-----|
| Cap3.1 Shell tokens + background | ✅ | [web #103](https://github.com/roamkit-net/roamkit-web/pull/103) |
| Cap3.2 TopBar + nav chrome | ✅ | [web #104](https://github.com/roamkit-net/roamkit-web/pull/104) |
| Cap3.3a Pilot `/me/esims` | ▶ open | [web #105](https://github.com/roamkit-net/roamkit-web/pull/105) |
| Cap3.3b Propagate | blocked on pilot staging accept | — |
| Cap3.4 CTA theme | after 3.3b | — |
| Cap3.5 Validation | last | — |

**Gates (do not skip):** Cap3.2 = TopBar/nav chrome only (no page cards). Cap3.3b only after `/me/esims` pilot accepted on staging; any surface-token fix lands once before propagate.  
Cap2 Merge leftovers = migration backlog, not Cap3.

## Inventory

[component-inventory.md](./component-inventory.md) (Cap2b — historical)
