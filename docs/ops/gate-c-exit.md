# Gate C exit tracker

Engineering checklist before **Gate C = GO**. See [Launch Gates](./launch-gates.md)
and [production-readiness-review.md](./production-readiness-review.md) conditions C1–C9.

| ID | Condition | Status | Evidence |
|----|-----------|--------|----------|
| C1 | Production settings | ⏳ api PR | `config.settings.production` fail-fast + HSTS |
| C3 | Deploy path + rollback | ⏳ | `deploy-production.sh` + `deploy-production.yml` on `main` |
| C4 | Secret scanning on `main` PRs | ⏳ | `secret-scan.yml` (Gitleaks) in api CI |
| C5 | `GET /version` | ⏳ api PR | non-empty `git_sha` in smoke |
| C6 | Billing E2E | ⏳ | `roamkit-infra/scripts/production-dod-billing.sh` |
| C7 | Sentry + uptime + reconcile alert | ⏳ | Sentry via `SENTRY_DSN`; uptime on health+/version; reconcile alert channel TBD |
| C8 | Backup + restore procedure | ⏳ | [migration-ready.md](./migration-ready.md); Disaster Day post-stable |
| C9 | Incident runbook | ✅ | [incident-runbook.md](./incident-runbook.md) |
| C10 | OpenAPI | N/A | deferred |

## Operator host steps (not closed by merge alone)

- [ ] Production stack bootstrapped (`init-production-stack.sh`)
- [ ] Real `.env` (no `change-me`); `SENTRY_DSN` set
- [ ] Migration Ready signed for first migrate
- [ ] Smoke: `smoke-test-production.sh` (requires `/version`)
- [ ] Billing DoD green (dress-rehearsal URLs OK before public DNS)
- [ ] Uptime monitors on `/health/live`, `/health/ready`, `/version`
- [ ] Reconcile-drift notification channel configured

**Verdict:** Engineering GO only when rows above are green and operator host steps checked.

## Related

- [Release Manifest draft](./releases/1.0.0.md)
- [Gate D cutover](./gate-d-cutover.md)
