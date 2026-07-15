# ADR 009: Shared Traefik edge routing

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze (revision) |
| Supersedes | Per-stack nginx + certbot in staging compose (Faza -1a draft) |

## Context

The initial `roamkit-infra` staging compose included a dedicated **nginx** container binding host ports 80/443 and local **certbot** volumes for TLS.

The Hetzner dedicated server already runs a **single shared Traefik** instance at `/opt/stacks/traefik/` (v3.6.7). All other projects (`stay.hr`, `fiskal.hr`, `racunai.hr`, etc.) attach to the external Docker network **`proxy`** and register routes via **Docker labels only**.

A per-stack nginx would:

- Conflict on ports 80/443 with Traefik
- Duplicate TLS management (Traefik already uses `certresolver=cloudflare`)
- Diverge from established server operations

Cloudflare connects via **orange cloud proxy** to origin IP `65.108.196.92:443`. Traefik terminates TLS on the host.

## Decision

**Staging stack** (`/opt/stacks/roamkit-net/`):

1. **No nginx** or certbot in RoamKit compose
2. **`api` and `web` services** join external network `proxy` with Traefik labels:

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=proxy
  - traefik.http.routers.roamkit-api.rule=Host(`api.staging.roamkit.net`)
  - traefik.http.routers.roamkit-api.entrypoints=websecure
  - traefik.http.routers.roamkit-api.tls=true
  - traefik.http.routers.roamkit-api.tls.certresolver=cloudflare
  - traefik.http.services.roamkit-api.loadbalancer.server.port=8000
```

3. **Internal services** use **shared** host PostGIS (`/opt/stacks/data`, network `postgis`) and Redis (`/opt/stacks/redis`, container `infra-redis`, network `hetzner_net`, DB index **4**). No per-stack DB/Redis containers.
4. **`api` / `celery`** join `postgis`, `hetzner_net`, and `proxy` (api only for Traefik); **`web`** joins `proxy` only
5. **RoamKit does not install Traefik** — prerequisite documented in `roamkit-infra/bootstrap/hetzner/prerequisites.md`
6. **Deploy SSH user:** `root` (matches WSL `ssh dedicated-hel1`); `STAGING_HOST=65.108.196.92` for GitHub Actions

Production (Faza 4) will follow the same Traefik pattern with production hostnames.

## Consequences

### Positive

- Consistent with all other stacks on the dedicated server
- No port conflicts; Traefik handles routing and TLS
- Simpler compose — fewer containers and volumes
- Cloudflare DNS-01 certs managed once at Traefik level

### Negative

- RoamKit deploy depends on shared Traefik uptime
- Label/router names must be unique across all stacks on the host (`roamkit-api`, `roamkit-web`)

### Application requirements

Django staging/production settings must trust proxy headers:

```python
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
USE_X_FORWARDED_HOST = True
```

## Related

- [ADR 006](./006-ghcr-pull-only-deploy.md)
- [ADR 007](./007-staging-only-until-launch.md)
- `roamkit-infra/bootstrap/hetzner/prerequisites.md`
- `roamkit-infra/docker/docker-compose.staging.yml`
