# CI Standard

GitHub Actions expectations for RoamKit repos. Workflow templates live in `roamkit-infra/ci/workflows/` and are copied to per-repo `.github/workflows/` or org `.github` as reusable workflows.

## Triggers

| Event | Jobs |
|-------|------|
| Pull request | Lint, test, security scan, Docker build (no deploy) |
| Push to `develop` | Above + deploy staging (api/web) |
| Push to `main` | Lint, test, security, build — **no deploy** |

## Per-repo pipelines

### roamkit-api (`ci-python.yml`)

1. **Lint** — ruff, black `--check`
2. **Test** — start `docker-compose.test.yml`, run pytest with coverage
3. **Security** — Bandit on `src/`
4. **Build** — Docker image build (on PR and main/develop)

### roamkit-web (`ci-node.yml`)

1. **Lint** — eslint
2. **Build** — `next build`
3. **Security** — dependency audit (npm audit or equivalent)
4. **Build** — Docker image (when Dockerfile exists)

### roamkit-infra

- Validate shell scripts (`shellcheck` when added).
- YAML lint on compose and workflow files.
- No application tests.

### roamkit-docs

- Markdown link check optional.
- No deploy.

## Docker build and push (`docker-build-push.yml`)

On merge to `develop`:

- Build image tagged with `github.sha`
- Push to `ghcr.io/roamkit-net/<repo>:<sha>`
- **Trivy** scan — fail on critical vulnerabilities without waiver

## Deploy staging (`deploy-staging.yml`)

Reusable workflow; runs only on `develop`:

1. SSH to `STAGING_HOST` (`65.108.196.92`) as **root** with `STAGING_SSH_KEY`
2. Deploy to `/opt/stacks/roamkit-net/`
2. Set `API_IMAGE` / `WEB_IMAGE` to GHCR SHA tags
3. Run `deploy-staging.sh` (pull, up, migrate, health, smoke)
4. Rollback on failure via `.previous-tag`
5. External smoke from runner: curl staging URLs

## Coverage

- pytest coverage threshold: start at 70% in Faza 0, raise to 80% by Faza 2.
- Coverage reports uploaded as CI artifacts.

## Secrets

Documented in `roamkit-infra/bootstrap/github/secrets.md`. Never log secrets in workflow output.

| Secret | Used by |
|--------|---------|
| `STAGING_HOST`, `STAGING_SSH_KEY` | deploy |
| `GHCR_TOKEN` | image push/pull |
| Repo-specific | Django, Airalo, Stripe |

## Failure policy

- PR checks must be green before merge.
- Failed staging deploy triggers rollback and should open/notify an issue.
- Security scan failures block merge unless ADR-documented exception.

## Related

- [ADR 006](../docs/adr/006-ghcr-pull-only-deploy.md)
- [ADR 009](../docs/adr/009-shared-traefik-edge.md)
- [Branching standard](./branching.md)
- [Definition of Done](./definition-of-done.md)
