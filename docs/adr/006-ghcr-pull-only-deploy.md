# ADR 006: GHCR pull-only deploy

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

Building Docker images on the Hetzner staging server couples deploy hosts to source code, build tools, and inconsistent layer caches. Failed builds on production-like hosts are hard to reproduce in CI.

## Decision

**CI builds and pushes** images to GitHub Container Registry (`ghcr.io/roamkit-net/...`).  
**Staging server only pulls** pre-built images and runs `docker compose up`.

Deploy flow (`roamkit-infra/scripts/deploy-staging.sh`):

1. Save current image tag to `.previous-tag` (rollback).
2. `docker compose pull`
3. `docker compose up -d`
4. `migrate`
5. Health: `/health/live`, `/health/ready`
6. Smoke test script
7. On failure: rollback to `.previous-tag`

Image tags: commit SHA on `develop` merges (e.g. `ghcr.io/roamkit-net/roamkit-api:${{ github.sha }}`).

The server **never** runs `docker build` for application images.

## Consequences

### Positive

- Build once in CI; same artifact promoted to staging.
- Faster deploys on modest VPS hardware.
- Trivy image scan runs in CI before push.

### Negative

- Requires GHCR auth on server and in Actions.
- Rollback depends on retained previous image tags in GHCR.

### Secrets

- `GHCR_TOKEN` at org level for pull/push.
- Runtime app secrets in server `.env`, not in image layers.

## Related

- [CI standard](../../standards/ci-standard.md)
- [ADR 007](./007-staging-only-until-launch.md)
- `roamkit-infra/ci/workflows/docker-build-push.yml`
