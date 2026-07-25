# Evidence — Gate C host check

Dress-rehearsal production stack bootstrap on shared HEL1 host.
**Does not** move apex/www DNS or Traefik Host traffic (Gate D).

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Gate | C |
| Milestone | Production host bootstrap |
| Verdict | GO (host stack up; public Traefik cutover deferred to Gate D) |
| Closed at (UTC) | 2026-07-25T21:38:20Z |
| GO by (role / name) | Engineering (solo operator) |
| API image | `ghcr.io/roamkit-net/roamkit-api:caa3f1d0e3e4c53bacf76719a0f18b1473560207` |
| Web image | `ghcr.io/roamkit-net/roamkit-web:bf62035325279c87153a93d71efc167a9bc5f80d` |
| Stack dir | `/opt/stacks/roamkit-production/` |
| DB | `roamkit_production` on shared PostGIS |
| Redis | `redis://infra-redis:6379/5` |

## Checklist performed

| Step | Result |
|------|--------|
| `init-production-stack.sh` | ✅ stack dir created |
| `CREATE DATABASE roamkit_production` | ✅ (as `postgres` superuser; `roamkit` lacked CREATEDB) |
| Real `.env` (no `change-me`) | ✅ new `DJANGO_SECRET_KEY`; staging PG/email/Airalo/wallet reused |
| Wallet key in `.secrets/` | ✅ copied from staging |
| `SENTRY_DSN` | ⏳ not set — blocks Observability evidence |
| `deploy-production.sh` | ✅ migrate + health + dress-rehearsal smoke |
| Staging apex/www undisturbed | ✅ production `traefik.enable=false` |

## Dress-rehearsal isolation (pre–Gate D)

To avoid Traefik Host collision with staging (`roamkit.net` / `www`):

- Production compose: `traefik.enable=false` (backup: `docker-compose.yml.bak-pre-dress`)
- Localhost publishes for ops checks: `127.0.0.1:18000→api`, `127.0.0.1:13000→web`
- Public HTTPS Host/DNS cutover remains Gate D

## Live `/version` (compose exec / localhost)

```json
{"git_sha": "caa3f1d0e3e4c53bacf76719a0f18b1473560207", "build_date": "2026-07-25T21:38:06Z", "image_tag": "caa3f1d0e3e4c53bacf76719a0f18b1473560207", "environment": "production"}
```

Matches Gate C API-slice merge SHA (`caa3f1d`).

## Containers (healthy)

- `roamkit-api-production`
- `roamkit-web-production`
- `roamkit-celery-production`
- `roamkit-celery-beat-production`

## Notes

- Airalo remains `AIRALO_SANDBOX=true` for dress-rehearsal (same partner sandbox as staging).
- Catalog synced: `sync_packages` → 1983 active packages.
- First migrate ran on empty DB during deploy; post-migrate backup recorded under Migration Ready.
