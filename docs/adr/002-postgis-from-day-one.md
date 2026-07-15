# ADR 002: PostGIS from day one

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

RoamKit may need geographic queries (coverage maps, country/region filters, future location-aware features). Migrating from plain PostgreSQL to PostGIS later is costly and risks data migration downtime.

Other projects in the same workspace already standardize on `postgis/postgis:16-3.4` for backup, compose, and operational familiarity.

## Decision

Use **PostGIS 16** (`postgis/postgis:16-3.4`) in all environments from day one:

- `docker-compose.dev.yml` (local WSL)
- `docker-compose.test.yml` (CI pytest)
- `docker-compose.staging.yml` (Hetzner)
- Future production stack

Django uses the PostGIS backend when spatial models are introduced. Until then, the database behaves as PostgreSQL with the PostGIS extension available.

## Consequences

### Positive

- No image swap or extension migration later.
- Consistent compose and backup procedures across dev, test, staging, production.
- Spatial indexes and geometry types available when product needs them.

### Negative

- Slightly larger image than `postgres:16`.
- Developers must use the PostGIS image locally (already in `roamkit-infra`).

### Requirements

- All compose files reference the same image tag family.
- `.env.example` documents `POSTGRES_*` variables; secrets never committed.

## Related

- [Docker standard](../../standards/docker-standard.md)
- `roamkit-infra/docker/docker-compose.dev.yml`
