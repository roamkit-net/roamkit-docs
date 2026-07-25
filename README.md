# roamkit-docs

Architecture decisions, RFCs, and engineering standards for [RoamKit](https://github.com/roamkit-net).

This repo is the **source of truth for documentation** — ADRs and standards are not scattered across `roamkit-api`, `roamkit-web`, or `roamkit-infra`.

## Contents

| Path | Purpose |
|------|---------|
| [ADR_INDEX.md](./ADR_INDEX.md) | Index of all Architecture Decision Records |
| [docs/architecture/](./docs/architecture/) | System overview, directory layout, provider patterns |
| [docs/adr/](./docs/adr/) | ADR 001–010 (accepted decisions) |
| [docs/rfcs/](./docs/rfcs/) | Request for Comments — proposed features before implementation |
| [standards/](./standards/) | Branching, coding, Docker, CI, API versioning, DoD |

## How to use

1. **Before implementing a feature** — check ADRs and relevant standards.
2. **When making an architectural change** — add a new ADR (next number), update `ADR_INDEX.md`, and link from affected standards.
3. **For larger product flows** — open an RFC first; promote to ADR once accepted.

## Related repos

| Repo | Role |
|------|------|
| `roamkit-infra` | Bootstrap, compose, CI templates, deploy scripts |
| `roamkit-api` | Django + DRF + Celery |
| `roamkit-web` | Next.js 15 App Router |
| `.github` | Org reusable workflows, PR template, CODEOWNERS |

## Status

Architecture Freeze (Faza -1b): ADR 001–009, standards, and Definition of Done are defined here. Faza 3 adds [ADR 010](./docs/adr/010-polygon-usdt-prepaid-credits.md) (Polygon USDT prepaid credits).
