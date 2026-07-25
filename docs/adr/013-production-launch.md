# ADR 013: Production launch

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Faza 4 gate opener |

## Context

[ADR 007](./007-staging-only-until-launch.md) kept `main` free of auto-deploy until an explicit production-launch decision. Faza 3 prepaid credits ([ADR 010](./010-polygon-usdt-prepaid-credits.md)) are complete on staging. Public launch needs accepted production URLs, a separate stack, live partner credentials, and a deploy trigger from `main`.

This ADR **opens** the Faza 4 gate. It does **not** provision infrastructure by itself — compose, secrets, CI deploy, and cutover land in subsequent infra/app PRs under the Go-Live checklist.

## Decision

### Deploy matrix (supersedes ADR 007)

| Branch | CI | Auto-deploy |
|--------|-----|-------------|
| `feature/*` | On PR (lint, test, security, build) | No |
| `develop` | Full pipeline | **Yes → staging** (`/opt/stacks/roamkit-net/`) |
| `main` | Full pipeline | **Yes → production** (`/opt/stacks/roamkit-production/`) once the production stack + `deploy-production` CI exist |

Promotion path: `feature/*` → `develop` (staging) → PR `develop` → `main` (production).

### URLs

| Environment | Web | API |
|-------------|-----|-----|
| **Production** | `https://roamkit.net`, `https://www.roamkit.net` | `https://api.roamkit.net` |
| **Staging** (after cutover) | `https://staging.roamkit.net` | `https://api.staging.roamkit.net` |

**Pre-cutover note:** Today `roamkit.net` / `www` may still route to the **staging** web service (marketing-on-staging). Cutover moves those hosts to the production web service; staging retains `staging.*` only. Do not drop staging Host rules for apex/www until cutover day.

### Stack isolation

| Concern | Rule |
|---------|------|
| Path | `/opt/stacks/roamkit-production/` — separate from `/opt/stacks/roamkit-net/` |
| Compose | Own `docker-compose.yml` (`profiles: [app]`); pull-only from GHCR ([ADR 006](./006-ghcr-pull-only-deploy.md)) |
| Edge | Shared Traefik on `proxy` ([ADR 009](./009-shared-traefik-edge.md)); production Host rules for apex/www/api |
| Database | **Separate PostGIS database** (not the staging DB name/schema) |
| Redis | **Separate Redis DB index** (not staging’s index) |
| Volumes | **No shared volumes** with staging |
| Secrets | Live credentials **only** in production `.env` (never in git) |

### Live credentials (production `.env` only)

| Integration | Production |
|-------------|------------|
| Airalo | **Production** Partner API credentials |
| Polygon USDT | Live `POLYGON_RPC_URL`, `POLYGON_PLATFORM_WALLET`, `POLYGON_USDT_CONTRACT`, `POLYGON_CHAIN_ID=137`, confirmations ([ADR 010](./010-polygon-usdt-prepaid-credits.md)) |
| Stripe / cards | **Not used** — prepaid credits only |

Staging may keep sandbox Airalo + non-production Polygon settings.

### Feature flags at first cutover

| Feature | Staging | Production (cutover) |
|---------|---------|----------------------|
| `BILLING_ENABLED` | ON | ON |
| `WALLETCONNECT_ENABLED` | ON (when AppKit smoked) | **OFF** at first cutover; re-enable after WalletConnect smoke |
| `SUBSCRIPTIONS_ENABLED` | OFF | OFF |
| `VOUCHERS_ENABLED` | OFF until voucher PR ship | OFF until voucher PR ship |
| Business / Team accounts | N/A (not built) | N/A |

Canonical matrix also lives in [production go-live checklist](../ops/production-go-live-checklist.md).

### Must-haves before declaring go-live

Required before customer traffic on production (not optional polish):

1. Architecture tests in CI (ADR 010 money-path boundaries).
2. Secret scanning on PRs / merges to `main` (and preferably all PRs).
3. End-to-end billing smoke (account → credit → buy → ledger → compensating refund path).
4. `GET /version` diagnostic endpoint (non-secret build metadata).

Full operational gate: [production go-live checklist](../ops/production-go-live-checklist.md).

### ADR 007 status

ADR 007 is **Superseded** by this ADR for the deploy-trigger and production URL decision. Staging continues to deploy from `develop`; only the “no `main` auto-deploy” rule is lifted under the conditions above.

## Consequences

### Positive

- Explicit, reviewable production cutover instead of accidental `main` deploys.
- Clear isolation rules reduce staging/prod data bleed.
- Billing launch criteria match ADR 010 (Polygon USDT), not obsolete Stripe language.

### Negative

- Apex domain cutover requires coordinated Traefik + DNS change.
- Production stack and CI must exist before `main` auto-deploy is wired (infra PR1–PR2).

### Implementation order (Faza 4)

1. This ADR + Go-Live checklist (**PR0**, docs) — done when Accepted.
2. Production readiness review + arch-test gaps (**PR0.5**).
3. Production compose / env / bootstrap (**PR1**, infra).
4. Deploy path, must-haves, cutover (**PR2**).
5. Observability (**PR3**), hardening (**PR4**).

Do **not** implement vouchers or enable subscriptions as part of production launch.

## Related

- [ADR 006](./006-ghcr-pull-only-deploy.md) — GHCR pull-only deploy
- [ADR 007](./007-staging-only-until-launch.md) — superseded staging-only deploy gate
- [ADR 009](./009-shared-traefik-edge.md) — shared Traefik edge
- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — Polygon USDT prepaid credits
- [Production go-live checklist](../ops/production-go-live-checklist.md)
- [Branching standard](../../standards/branching.md)
- [ADR_INDEX](../../ADR_INDEX.md)
