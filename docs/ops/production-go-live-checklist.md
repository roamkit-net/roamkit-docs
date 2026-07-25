# Production go-live checklist

Operational gate for Faza 4 cutover. Architecture: [ADR 013](../adr/013-production-launch.md).  
Do **not** declare go-live until every applicable box is checked.

## Feature flag matrix

| Feature | Env / flag | Staging | Production (first cutover) |
|---------|------------|---------|----------------------------|
| Billing | `BILLING_ENABLED` | ON | ON |
| WalletConnect | `WALLETCONNECT_ENABLED` | ON after AppKit smoke | **OFF** (re-enable after WC smoke on prod) |
| Subscriptions | `SUBSCRIPTIONS_ENABLED` | OFF | OFF |
| Vouchers | `VOUCHERS_ENABLED` | OFF until voucher ship | OFF until voucher ship |
| Business / Team | — | not built | not built |

Record actual prod `.env` values at cutover in the sign-off section below.

## PR0 — Decision gate

- [x] [ADR 013](../adr/013-production-launch.md) Accepted
- [x] [ADR 007](../adr/007-staging-only-until-launch.md) marked Superseded for deploy trigger
- [x] This checklist + flag matrix published
- [x] `ADR_INDEX` / branching / `AGENTS.md` synced (same PR as ADR 013; app repos get matching `AGENTS.md` PRs)

## PR0.5 — Production readiness review

- [ ] All ADRs in [ADR_INDEX](../../ADR_INDEX.md) are Accepted or explicitly Superseded
- [ ] No `TODO`/`FIXME` in money-path / auth / deposit code (scoped grep)
- [ ] Production settings: no debug noise defaults; no hardcoded development API keys
- [ ] Feature flags documented (matrix above + settings names)
- [ ] OpenAPI / schema export checked against backend (or documented N/A)
- [ ] ADR 010 architecture tests green and **blocking** on CI for `develop` and `main`
- [ ] **Signed off by:** _____________ **Date:** _____________

## PR1 — Production infrastructure

- [ ] `/opt/stacks/roamkit-production/` initialized
- [ ] `docker-compose.production.yml` present (api, celery, beat, web; `profiles: [app]`)
- [ ] Traefik labels for `roamkit.net` \|\| `www.roamkit.net` and `api.roamkit.net`
- [ ] Separate PostGIS database + separate Redis DB index documented
- [ ] No shared volumes with staging
- [ ] `.env.production.example` committed; real `.env` on host only
- [ ] Secrets: `PRODUCTION_HOST`, `PRODUCTION_SSH_KEY`, GHCR pull documented
- [ ] Cutover note: staging Host rules drop apex/www **only at cutover**

## PR2 — Deploy path + must-haves

### Must-haves ([ADR 013](../adr/013-production-launch.md))

- [ ] (1) Architecture tests blocking in CI
- [ ] (2) Secret scanning (GitHub + Gitleaks/TruffleHog) required on `main` PRs
- [ ] (3) Billing E2E smoke script green on production (or staging dress-rehearsal then prod)
- [ ] (4) `GET /version` returns 200 with non-empty `git_sha`

### Deploy

- [ ] `deploy-production.sh` (pull → up → migrate → health → smoke; rollback via `.previous-tag`)
- [ ] CI `deploy-production.yml` on `main` only
- [ ] Real `config.settings.production` (`DEBUG=False`, strict hosts/CSRF/cookies/HSTS)
- [ ] Web build: `NEXT_PUBLIC_API_URL=https://api.roamkit.net`
- [ ] DNS: `api.roamkit.net` → origin; apex/www cutover planned
- [ ] Traefik LE certs via `certresolver=cloudflare`
- [ ] HTTP smoke + billing DoD after cutover
- [ ] Staging remains on `staging.*` only after apex/www move

### Billing E2E smoke (minimum)

```text
Create account → Deposit/credit → Verify → Balance +N
→ Buy eSIM → Balance -price → Ledger check
→ Refund/compensating path → Balance restored
```

- [ ] Script path: `roamkit-infra/scripts/production-dod-billing.sh` (or documented equivalent)
- [ ] Last green run: **SHA** _____________ **Date** _____________

## PR3 — Observability

- [ ] Structured logs (API/Celery/web) with request/correlation id where feasible
- [ ] Sentry for api + web (DSN only in prod `.env`)
- [ ] Uptime on `/health/live`, `/health/ready`, web origin, `/version`
- [ ] Alerting: uptime + Sentry + **billing reconcile drift** (no auto balance correction)
- [ ] Incident runbooks under `docs/ops/`

## PR4 — Hardening + exercises

- [ ] Rate limits on auth + billing verify; security headers
- [ ] Backup **restore** test recorded (not only backup job exists)
- [ ] **Disaster Day** notes filed (Postgres / Redis / API / web / Traefik restart; recovery time; rollback path)
- [ ] Lean load test recorded (deposit verify / order / balance); no 5xx storms
- [ ] Dependency audit in deploy CI (`pip-audit`, `npm audit --omit=dev`); critical fails job
- [ ] Release notes on prod deploy (version, commits, migrations, rollback SHA)
- [ ] Perf budget warnings baseline (optional fail later)
- [ ] SEO/error pages: `robots.txt`, sitemap, 404/500 as applicable

## Cutover day

- [ ] Feature flag matrix matches running prod `.env` (paste values below)
- [ ] Admin account accessible
- [ ] Deposit + buy manually verified once
- [ ] Log rotation / disk monitoring OK
- [ ] Rollback owner + command documented for the window

### Prod `.env` flag snapshot (cutover)

```text
BILLING_ENABLED=
WALLETCONNECT_ENABLED=
SUBSCRIPTIONS_ENABLED=
VOUCHERS_ENABLED=
```

## Go-live sign-off

Faza 4 go-live is complete when PR0–PR4 scopes above are done, PR0.5 is signed, four must-haves are green on production, Disaster Day notes exist, and this checklist is fully checked.

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Operator | | | |
| Engineering | | | |

## Related

- [ADR 013](../adr/013-production-launch.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [ADR 007](../adr/007-staging-only-until-launch.md) (Superseded)
- [Definition of Done](../../standards/definition-of-done.md)
