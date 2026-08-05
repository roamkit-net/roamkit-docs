# Design System status

RoamKit web primitives (ADR 016). Source of truth for Cap2 rollout.

Colocated mirror: `roamkit-web/components/ui/README.md`.

| Primitive | Status | Canonical | PR |
|-----------|--------|-----------|-----|
| Button | ✅ | `ui/Button` | [#97](https://github.com/roamkit-net/roamkit-web/pull/97) |
| Alert | ✅ | `ui/Alert` | [#98](https://github.com/roamkit-net/roamkit-web/pull/98) |
| Input | ⏳ | `ui/Input` | Cap2.3 — [#99](https://github.com/roamkit-net/roamkit-web/pull/99) |
| Card | ⏳ | `ui/Card` | Cap2.4 |
| Skeleton | ⏳ | `ui/Skeleton` | Cap2.5 |
| Empty | ⏳ | `ui/EmptyState` | Cap2.5 |
| Badge | ⏳ | `ui/Badge` | Cap2.5 |

## Rules

- Brand → Theme → Components → Pages ([ADR 016](../adr/016-web-design-tokens.md))
- Migration: create primitive → tests → migrate callers → delete duplicates
- Cap2 stop rule: architecture only — no intentional visual change
- Alert ≠ Toast

## Inventory

[component-inventory.md](./component-inventory.md) (Cap2b)
