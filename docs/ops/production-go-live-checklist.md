# Production go-live checklist

Operational gate for Faza 4 cutover. Architecture: [ADR 013](../adr/013-production-launch.md).  
Release language: [Launch Gates](./launch-gates.md) (Gate C → Gate D). Handbook: [ops README](./README.md).  
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

- [x] Go/no-go review published: [production-readiness-review.md](./production-readiness-review.md)
- [x] Verdict: **GO WITH CONDITIONS** (PR1 infra authorized; cutover blocked on C1–C9)
- [x] All ADRs in [ADR_INDEX](../../ADR_INDEX.md) are Accepted or explicitly Superseded
- [x] No `TODO`/`FIXME` in money-path / auth / deposit code (api arch CI guard)
- [ ] Production settings: no debug noise defaults; no hardcoded development API keys (**C1** — PR2)
- [x] Feature flags documented (matrix above + readiness review)
- [x] OpenAPI: schema + CI + staging docs (**C10** closed) — [openapi-c10.md](./releases/1.0.0/evidence/openapi-c10.md)
- [x] ADR 010 architecture tests green and **blocking** on CI for `develop` (extend on `main` when prod CI exists)
- [x] **Signed off by:** Engineering/Operator — **GO WITH CONDITIONS** — **Date:** 2026-07-25

Full scorecard and conditions: [production-readiness-review.md](./production-readiness-review.md).

## PR1 — Production infrastructure

Infra only (`roamkit-infra`). No `roamkit-api` / `roamkit-web` code in this PR.

- [x] `/opt/stacks/roamkit-production/` init script (`init-production-stack.sh`)
- [x] `docker-compose.production.yml` (api, celery, beat, web; `profiles: [app]`)
- [x] Traefik labels for `roamkit.net` \|\| `www.roamkit.net` and `api.roamkit.net`
- [x] Separate PostGIS database name + Redis DB index documented (`roamkit_production` / `/5`)
- [x] No shared volumes with staging (stack isolation)
- [x] `.env.production.example` committed; real `.env` on host only
- [x] Secrets: `PRODUCTION_HOST`, `PRODUCTION_SSH_KEY`, GHCR + lifecycle in PRODUCTION_PLAN
- [x] Cutover note: staging Host rules drop apex/www **only at cutover**
- [x] Deploy / rollback / smoke scripts + health policy + startup order
- [x] Capability matrix: [capability-status.md](./capability-status.md)

Operator still must run bootstrap on the host and complete the merge checklist in `PRODUCTION_PLAN.md` before treating the platform as live.

## PR2 — Deploy path + must-haves

### Must-haves ([ADR 013](../adr/013-production-launch.md))

- [ ] (1) Architecture tests blocking in CI
- [ ] (2) Secret scanning (GitHub + Gitleaks/TruffleHog) required on `main` PRs
- [x] (3) Billing E2E smoke script green on production (or staging dress-rehearsal then prod) — public PASS 2026-07-26 [billing-e2e-criterion-3.md](./releases/1.0.0/evidence/billing-e2e-criterion-3.md)
- [ ] (4) `GET /version` returns 200 with non-empty `git_sha`

### Deploy

- [ ] `deploy-production.sh` (pull → up → migrate → health → smoke; rollback via `.previous-tag`)
- [ ] CI `deploy-production.yml` on `main` only
- [ ] Real `config.settings.production` (`DEBUG=False`, strict hosts/CSRF/cookies/HSTS)
- [x] Web build: `NEXT_PUBLIC_API_URL=https://api.roamkit.net` — Criterion #1 (`7064a9b…` main rebuild digest `732856f0…`)
- [x] Web + API `GET /version` return non-empty `git_sha` — API public `/version`; web via bake/deploy evidence
- [x] DNS: `api.roamkit.net` → origin; apex/www on production
- [x] Traefik LE certs via `certresolver=cloudflare` — production Hosts live
- [x] HTTP smoke + billing DoD after cutover — public curls + Playwright catalog; Billing DoD Criterion #3
- [x] Staging remains on `staging.*` only after apex/www move

### Billing E2E smoke (minimum)

```text
Create account → Deposit/credit → Verify → Balance +N
→ Buy eSIM → Balance -price → Ledger check
→ Refund/compensating path → Balance restored
```

Catalog price acceptance (same cutover window):

```text
Open /global-esim → catalog-price-skeleton gone → prices visible
→ Network GET /api/v1/billing/config/ 200 on api.roamkit.net
```

- [x] Script path: `roamkit-infra/scripts/production-dod-billing.sh` (or documented equivalent)
- [x] `/global-esim` prices visible (no sticky skeleton); config host is production API — Playwright PASS 2026-07-26 ([gate-d-criterion-1.md](./releases/1.0.0/evidence/gate-d-criterion-1.md))
- [x] Last green run: **SHA** `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` **Date** `2026-07-26T11:46:20Z` (public `https://api.roamkit.net`)

## PR3 — Observability

- [ ] Structured logs (API/Celery/web) with request/correlation id where feasible *(backlog; not Criterion #2 hard DoD)*
- [x] Sentry for api + web (DSN only in prod `.env`) — Criterion #2 PASS 2026-07-26
- [x] Uptime on `/health/live`, `/health/ready`, web origin, `/version`, **`/api/v1/billing/config/`** — Kuma 13–17
- [x] Alerting: uptime + Sentry + **billing reconcile drift** (no auto balance correction) — Sentry event + drift channel; Kuma UI
- [x] Incident runbooks under `docs/ops/` — includes Observability monitors section

Evidence: [observability-criterion-2.md](./releases/1.0.0/evidence/observability-criterion-2.md).

## PR4 — Hardening + exercises

- [ ] Rate limits on auth + billing verify; security headers
- [ ] Backup **restore** test recorded (not only backup job exists)
- [ ] Backup restore procedure documented + backup-before-migrate ([migration-ready.md](./migration-ready.md)); full [Disaster Day](./disaster-day.md) after stable prod (annual; not Gate D hard blocker)
- [ ] Lean load test recorded (deposit verify / order / balance); no 5xx storms
- [ ] Dependency audit in deploy CI (`pip-audit`, `npm audit --omit=dev`); critical fails job
- [ ] Release notes on prod deploy (version, commits, migrations, rollback SHA)
- [ ] Perf budget warnings baseline (optional fail later)
- [ ] SEO/error pages: `robots.txt`, sitemap, 404/500 as applicable

## Cutover day

- [ ] Feature flag matrix matches running prod `.env` (paste values below)
- [ ] Admin account accessible
- [ ] Deposit + buy manually verified once
- [ ] `/global-esim`: prices visible; no `catalog-price-skeleton`; config requests hit `api.roamkit.net`
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
