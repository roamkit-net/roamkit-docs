# Gate C exit tracker

Engineering checklist before **Gate C = GO**. See [Launch Gates](./launch-gates.md)
and [production-readiness-review.md](./production-readiness-review.md) conditions C1–C9.

| ID | Condition | Status | Evidence |
|----|-----------|--------|----------|
| C1 | Production settings | ✅ api merged | `config.settings.production` fail-fast + HSTS |
| C3 | Deploy path + rollback | ✅ scripts + workflow | `deploy-production.sh` + `deploy-production.yml` on `main` |
| C4 | Secret scanning on PRs | ✅ | Gitleaks CLI in api CI |
| C5 | `GET /version` | ✅ api merged | non-empty `git_sha` required in smoke |
| C6 | Billing E2E | ✅ script | `roamkit-infra/scripts/production-dod-billing.sh` (run on host) |
| C7 | Sentry + uptime + reconcile alert | ✅ | [observability.md](./releases/1.0.0/evidence/observability.md) |
| C8 | Backup + restore procedure | ✅ signed | [migration-ready.md](./migration-ready.md); Disaster Day post-stable |
| C9 | Incident runbook | ✅ | [incident-runbook.md](./incident-runbook.md) |
| C10 | OpenAPI | N/A | deferred |

## Operator host steps (not closed by merge alone)

- [x] Production stack bootstrapped (`init-production-stack.sh`) — 2026-07-25; [host-check.md](./releases/1.0.0/evidence/host-check.md)
- [x] Real `.env` (no `change-me`); `SENTRY_DSN` set — 2026-07-25
- [x] Migration Ready signed for first migrate — [migration-ready.md](./migration-ready.md)
- [x] Smoke: `smoke-test-production.sh` (requires `/version`) — dress-rehearsal ✅; [smoke.md](./releases/1.0.0/evidence/smoke.md)
- [x] Billing DoD green (dress-rehearsal URLs OK before public DNS) — [billing-e2e.md](./releases/1.0.0/evidence/billing-e2e.md)
- [x] Uptime monitors on `/health/live`, `/health/ready`, `/version` — Uptime Kuma ids 13–15 (dress-rehearsal); [observability.md](./releases/1.0.0/evidence/observability.md)
- [x] Reconcile-drift notification channel configured — beat + ERROR logs + Sentry production project

**Verdict:** **GO** — 2026-07-25T21:52:00Z. Evidence index: [gate-c.md](./releases/1.0.0/evidence/gate-c.md).

While those host steps run, [Gate C focus](./launch-gates.md#gate-c-focus-before-host-execution) applies (no parallel feature release work).

## Related

- [Release Manifest draft](./releases/1.0.0/manifest.md)
- [Gate C evidence index](./releases/1.0.0/evidence/gate-c.md)
- [Release Decision Log](./release-decision-log.md)
- [Gate D cutover](./gate-d-cutover.md)
