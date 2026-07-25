# Production Readiness Review (PR0.5)

| Field | Value |
|-------|-------|
| Date | 2026-07-25 |
| Scope | Faza 4 gate after [ADR 013](../adr/013-production-launch.md) |
| Reviewer | Solo operator (RoamKit) |
| Related | [Go-live checklist](./production-go-live-checklist.md) |

This is a **go / no-go** review, not a generic checklist dump. Statuses reflect
**evidence in the repos and staging today**, not aspirational Faza 4 work.

**Legend:** ✅ ready · ⚠️ condition / gap before go-live · ❌ blocker for starting Faza 4 infra

---

## Verdict

| Decision | Choice |
|----------|--------|
| **GO WITH CONDITIONS** | ✅ selected |
| GO | — |
| NO-GO | — |

**Meaning:** Faza 3 billing platform and ADR 013 decision gate are solid enough to
**start PR1 (production infra)**. Customer-facing cutover is **not** authorized
until the conditions below are closed (mostly PR2–PR4 must-haves).

### Conditions (must close before go-live)

| # | Condition | Owner PR |
|---|-----------|----------|
| C1 | Real `config.settings.production` (not staging stub); strict hosts/CSRF/HSTS | PR2 |
| C2 | `/opt/stacks/roamkit-production/` compose + separate DB/Redis + secrets | PR1 — **platform files landed**; host bootstrap + first deploy still operator |
| C3 | `deploy-production.sh` + `main` deploy CI + `.previous-tag` rollback | PR2 |
| C4 | Secret scanning required on `main` PRs (Gitleaks / org scanning) | PR2 |
| C5 | `GET /version` live + smoke assertion | PR2 |
| C6 | Production billing E2E smoke green (`production-dod-billing.sh`) | PR2 |
| C7 | Sentry + uptime + reconcile-drift alerting | PR3 |
| C8 | Backup restore test + Disaster Day notes | PR4 |
| C9 | Incident runbooks under `docs/ops/` | PR3 |
| C10 | OpenAPI: either ship schema export in CI **or** keep explicit N/A until first public contract freeze | PR2 / deferred |

**None of C1–C10 are reasons to delay PR1.** They are reasons to delay **cutover**.

---

## Scorecard

| Područje | Status | Napomena |
|----------|--------|----------|
| Architecture | ✅ | ADR 010–013 Accepted; 007 Superseded; hierarchy locked |
| Database | ⚠️ | Staging PostGIS OK; **prod DB not provisioned** (PR1). Migrations + CHECKs exist |
| Billing invariants | ✅ | `CreditService` sole mutator; arch tests; staging-dod-billing validated |
| Feature flags | ✅ | Matrix in ADR 013 + go-live checklist; settings in `base.py` |
| Monitoring | ⚠️ | Health live/ready only; **no Sentry / structured prod logging yet** (PR3) |
| Alerts | ⚠️ | Reconcile command/task exists; **no pager/alert routing** (PR3) |
| Backup / Restore | ⚠️ | Policy stub only; **restore not proven** (PR4) |
| Rollback | ⚠️ | Staging pattern proven (`.previous-tag`); **prod script missing** (PR2) |
| Security | ⚠️ | Bandit in CI; staging hardened (`DEBUG=False`); **no secret-scan on main**; prod settings stub |
| Documentation | ✅ | ADR 013, go-live checklist, billing DoD, staging billing README |
| Operations | ⚠️ | Staging smoke/DoD OK; **no prod ownership runbook / on-call notes** (PR3) |
| OpenAPI | ⚠️ | **No DRF spectacular/schema CI today** — documented gap (C10) |
| TODO / FIXME money path | ✅ | Scoped grep clean; CI arch guard added in api PR0.5 |

---

## Architecture

| Check | Status | Evidence |
|-------|--------|----------|
| ADR 010 money path Accepted | ✅ | `docs/adr/010-…` |
| ADR 012 extensibility Accepted | ✅ | constitution for credit sources |
| ADR 011 vouchers Accepted (design only) | ✅ | **not** required for prod cutover |
| ADR 013 production launch Accepted | ✅ | opens `main` deploy gate |
| ADR 007 Superseded | ✅ | deploy trigger moved to 013 |
| Arch tests in pytest suite | ✅ | `tests/test_billing_architecture.py` (blocking with suite) |
| Airalo ↛ billing | ✅ | arch test |
| Polygon ↛ billing money | ✅ | arch test |
| Only `CreditService` writes `Account.balance` | ✅ | arch test |
| Ledger append-only | ✅ | model + arch test |

**Gap closed in PR0.5 (api):** TODO/FIXME scan on money-path dirs; billing must not import frontend packages.

---

## Database

| Check | Status | Evidence |
|-------|--------|----------|
| Staging PostGIS healthy | ✅ | shared `/opt/stacks/data`, DB `roamkit` |
| Billing migrations + CHECKs | ✅ | `0001_billing_schema` |
| `Order.account` ownership | ✅ | migrated; ADR 010 |
| Production separate database | ⚠️ | **not created** — PR1 |
| Production separate Redis index | ⚠️ | **not assigned** — PR1 |
| No shared volumes staging↔prod | ⚠️ | rule in ADR 013; enforce at PR1 |

---

## Billing invariants

| Check | Status | Evidence |
|-------|--------|----------|
| Ledger SoT / balance cache | ✅ | ADR 010 + CreditService |
| Staging billing DoD | ✅ | `staging-dod-billing.sh` merged ([infra #9](https://github.com/roamkit-net/roamkit-infra/pull/9)) |
| Deposit → verify → ledger → order | ✅ | validated on staging per infra PR |
| Compensating REFUND on fulfill fail | ✅ | OrderService / TopupService |
| Airalo has zero billing knowledge | ✅ | arch test |

---

## Feature flags

Canonical cutover matrix ([ADR 013](../adr/013-production-launch.md)):

| Flag | Settings name | Staging target | Prod cutover |
|------|---------------|----------------|--------------|
| Billing | `BILLING_ENABLED` | ON | ON |
| WalletConnect | `WALLETCONNECT_ENABLED` | ON after AppKit smoke | **OFF** first cutover |
| Subscriptions | `SUBSCRIPTIONS_ENABLED` | OFF | OFF |
| Vouchers | `VOUCHERS_ENABLED` | not in settings until PR1 vouchers | OFF |

| Check | Status | Evidence |
|-------|--------|----------|
| Flags documented | ✅ | ADR 013 + go-live checklist |
| Defaults safe for accidental prod boot | ⚠️ | `BILLING_ENABLED` defaults **true** — prod `.env` must set explicitly; WalletConnect defaults false ✅ |
| `VOUCHERS_ENABLED` | ✅ N/A | not shipped; do not invent for cutover |

---

## Monitoring & alerts

| Check | Status | Evidence |
|-------|--------|----------|
| `/health/live`, `/health/ready` | ✅ | core health app; staging deploy curls them |
| Sentry (api/web) | ⚠️ | **absent** — PR3 |
| Uptime checks | ⚠️ | **absent** — PR3 |
| Billing reconcile job | ✅ | Celery beat + management command |
| Drift → human alert (no auto-fix) | ⚠️ | event exists; **no notification channel** — PR3 |
| `GET /version` | ⚠️ | **absent** — PR2 must-have |

---

## Backup / restore / rollback

| Check | Status | Evidence |
|-------|--------|----------|
| Staging deploy rollback on ERR | ✅ | `deploy-staging.sh` + `.previous-tag` |
| Production deploy rollback | ⚠️ | script not written — PR2 |
| DB backup job | ⚠️ | not evidenced in repo — PR4 |
| Restore test recorded | ⚠️ | required before go-live — PR4 |
| Disaster Day | ⚠️ | required before go-live — PR4 |

---

## Security

| Check | Status | Evidence |
|-------|--------|----------|
| Bandit in API CI | ✅ | `ci-python.yml` |
| Staging `DEBUG=False` | ✅ | `settings/staging.py` |
| Production settings module | ⚠️ | **stub** `from .staging import *` — PR2 must harden hosts/CSRF for `api.roamkit.net` |
| Secret scanning on `main` | ⚠️ | **not enabled** — PR2 must-have |
| No hardcoded live API keys in git | ✅ | env-driven; templates use placeholders |
| Default `DJANGO_SECRET_KEY` fallback | ⚠️ | exists for local — **prod `.env` must override**; fail-fast in production settings recommended (PR2) |

---

## Documentation & operations

| Check | Status | Evidence |
|-------|--------|----------|
| Go-live checklist | ✅ | `docs/ops/production-go-live-checklist.md` |
| Branching / AGENTS Faza 4 | ✅ | synced across repos |
| Staging billing ops docs | ✅ | infra README + secrets.md |
| Production incident runbook | ⚠️ | missing — PR3 |
| Ownership | ✅ | Solo operator; merge authority in AGENTS.md |
| Sign-off criteria | ✅ | this review + go-live checklist |

### Ownership (current)

| Area | Owner |
|------|-------|
| Architecture / ADR | Solo maintainer |
| Staging ops | Solo maintainer (`dedicated-hel1`) |
| Production cutover | Solo maintainer (same host, separate stack) |
| Billing money path | `apps.billing` + ADR 010/012 |
| On-call | Solo (document escalation in PR3 runbook) |

---

## OpenAPI / schema gate

| Check | Status | Note |
|-------|--------|------|
| Generated OpenAPI in CI | ⚠️ | **Not implemented** (no spectacular / schema job) |
| Decision | Documented N/A for PR0.5 | Add schema export when freezing public billing contract for external clients; internal Next.js types already consume REST. Revisit in PR2 if partner/OpenAPI consumers appear |

---

## Sign-off

| Role | Decision | Name | Date |
|------|----------|------|------|
| Engineering / Operator | **GO WITH CONDITIONS** | | 2026-07-25 |

**Authorized next step:** start **PR1 — production infrastructure** (`docker-compose.production`, env example, bootstrap).  
**Not authorized:** DNS/Traefik apex cutover or customer traffic until C1–C9 closed (C10 per policy above).

When conditions clear, re-run this scorecard and upgrade verdict to **GO**, then complete the [go-live checklist](./production-go-live-checklist.md) sign-off table.

## Related

- [ADR 013](../adr/013-production-launch.md)
- [Go-live checklist](./production-go-live-checklist.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [Definition of Done](../../standards/definition-of-done.md)
