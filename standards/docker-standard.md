# Docker Standard

Compose stacks and container conventions for RoamKit.

## Image sources

| Service | Image | All compose files |
|---------|-------|-------------------|
| Database | `postgis/postgis:16-3.4` | dev, test, staging |
| Cache/broker | `redis:7-alpine` | dev, test, staging |
| Mail (dev/test) | `axllent/mailpit:latest` | dev, test |

See [ADR 002](../docs/adr/002-postgis-from-day-one.md).

## Compose files (`roamkit-infra/docker/`)

| File | Use |
|------|-----|
| `docker-compose.dev.yml` | Local WSL — infra services only |
| `docker-compose.test.yml` | CI pytest — same DB/Redis images as dev |
| `docker-compose.staging.yml` | Hetzner `/opt/stacks/roamkit-net/` |

Application images (`roamkit-api`, `roamkit-web`) are referenced in staging compose; built in CI, not on server.

## Local development

```bash
cd roamkit-infra/docker
cp .env.example .env   # local only — never commit
docker compose -f docker-compose.dev.yml up -d
```

- App processes run on **host** (WSL): `runserver`, Celery, `npm run dev`.
- Code path: `~/projects/roamkit-net/`, not `/mnt/c/`.
- Verify stack: `roamkit-infra/bootstrap/docker/verify-local.sh`.

## Environment variables

- `.env.example` documents required keys with safe placeholders.
- Real secrets in `.env` (local) or `/opt/stacks/roamkit-net/.env` (server).
- Never commit `.env` or embed secrets in Dockerfiles.

## Application Dockerfiles (api/web)

When added in Faza 0:

- Multi-stage builds where practical (smaller runtime image).
- Non-root user in production stage.
- No dev dependencies in production image.
- Health check compatible with `GET /health/live`.

## Staging server

- Path: `/opt/stacks/roamkit-net/`
- **Shared data:** PostGIS at `/opt/stacks/data` (network `postgis`), Redis at `/opt/stacks/redis` (`infra-redis`, network `hetzner_net`, DB **4**)
- **App only:** `docker compose --profile app up -d` (api, celery, web — no local postgis/redis)
- Operations: `pull` → `up` → `migrate` — see `scripts/deploy-staging.sh`
- **No `docker build`** on Hetzner for app images ([ADR 006](../docs/adr/006-ghcr-pull-only-deploy.md)).
- No per-stack data volumes — DB/Redis are shared host services.

## Edge routing (Traefik)

Staging does **not** run per-stack nginx. Routing and TLS are handled by the **shared host Traefik** at `/opt/stacks/traefik/`.

RoamKit `api` and `web` containers:

- Join external Docker network **`proxy`**
- Register via Traefik Docker labels (`certresolver=cloudflare`)
- Hostnames: `staging.roamkit.net`, `api.staging.roamkit.net`

See [ADR 009](../docs/adr/009-shared-traefik-edge.md) and `roamkit-infra/bootstrap/hetzner/prerequisites.md`.

SSH from WSL: `ssh dedicated-hel1` (root@65.108.196.92). CI deploy uses the same user.

## Related

- [CI standard](./ci-standard.md)
- `roamkit-infra/docker/.env.example`
