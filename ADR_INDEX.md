# Architecture Decision Records — Index

All ADRs live in [docs/adr/](./docs/adr/). Status values: **Accepted**, **Superseded**, **Deprecated**.

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [001](./docs/adr/001-repo-split.md) | Multi-repo split (api, web, infra, docs) | Accepted | 2026-07 |
| [002](./docs/adr/002-postgis-from-day-one.md) | PostGIS from day one | Accepted | 2026-07 |
| [003](./docs/adr/003-src-layout.md) | Django `src/` layout | Accepted | 2026-07 |
| [004](./docs/adr/004-provider-interfaces.md) | Provider interfaces for eSIM integrations | Accepted | 2026-07 |
| [005](./docs/adr/005-domain-events.md) | In-process domain event bus | Accepted | 2026-07 |
| [006](./docs/adr/006-ghcr-pull-only-deploy.md) | GHCR pull-only deploy (no server builds) | Accepted | 2026-07 |
| [007](./docs/adr/007-staging-only-until-launch.md) | Staging-only deploy until production launch | Accepted | 2026-07 |
| [008](./docs/adr/008-bootstrap-iac-gh-cli.md) | Bootstrap IaC via `gh` CLI | Accepted | 2026-07 |
| [009](./docs/adr/009-shared-traefik-edge.md) | Shared Traefik edge routing (proxy network) | Accepted | 2026-07 |
| [010](./docs/adr/010-polygon-usdt-prepaid-credits.md) | Polygon USDT prepaid credits | Accepted | 2026-07 |

## RFCs (proposals)

| RFC | Title | Status |
|-----|-------|--------|
| [001](./docs/rfcs/001-self-service-esim-flow.md) | Self-service eSIM purchase and top-up flow | Draft |

## Architecture reference

- [System overview](./docs/architecture/overview.md)
- [Directory structure](./docs/architecture/directory-structure.md)
- [Provider abstractions](./docs/architecture/provider-abstractions.md)

## Standards

- [Branching](./standards/branching.md)
- [Python coding standard](./standards/coding-standard-python.md)
- [TypeScript coding standard](./standards/coding-standard-typescript.md)
- [Docker standard](./standards/docker-standard.md)
- [CI standard](./standards/ci-standard.md)
- [API versioning](./standards/api-versioning.md)
- [Definition of Done](./standards/definition-of-done.md)
