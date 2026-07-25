# Definition of Done

A work item (issue/PR) is **Done** when all applicable criteria below are met. Skip sections that do not apply (e.g. web-only doc change).

## Code and review

- [ ] Implementation matches acceptance criteria in the issue.
- [ ] PR targets `develop` (or `main` for release/hotfix per [branching](./branching.md)).
- [ ] At least one review approval (or solo maintainer self-review documented for docs-only).
- [ ] No unrelated changes in the PR.
- [ ] Follows [Python](./coding-standard-python.md) or [TypeScript](./coding-standard-typescript.md) standards as appropriate.

## Tests

- [ ] New behavior covered by automated tests (api: pytest; web: lint + build minimum).
- [ ] Existing tests pass locally and in CI.
- [ ] Coverage does not drop below CI threshold (api).

## Security and secrets

- [ ] No secrets, API keys, or `.env` files committed.
- [ ] Bandit (api) and dependency scans pass in CI.
- [ ] User input validated; authz checked on protected endpoints.

## Infrastructure and deploy

- [ ] If compose or deploy scripts changed: tested locally or on staging.
- [ ] Docker images build in CI when Dockerfile touched.
- [ ] Database migrations included and reversible when feasible.
- [ ] Health endpoints still pass after api changes.

## Documentation

- [ ] ADR or RFC updated if architectural decision changed.
- [ ] README or inline docs updated for new env vars, commands, or setup steps.
- [ ] API changes reflected in versioning notes or OpenAPI when applicable.

## Staging verification (features touching api/web)

- [ ] Merged to `develop` and deploy pipeline green.
- [ ] `GET /health/live` and `GET /health/ready` pass on staging.
- [ ] Smoke test script passes (`roamkit-infra/bootstrap/hetzner/smoke-test.sh`).
- [ ] Feature manually verified on `staging.roamkit.net` / `api.staging.roamkit.net`.

## Rollback awareness

- [ ] Deploy uses GHCR SHA tags; previous tag file written for rollback.
- [ ] Known migration risks documented in PR description.

## Phase-specific gates

| Phase | Additional Done criteria |
|-------|--------------------------|
| Faza 0 | Skeleton deploy shows "staging OK"; health endpoints live |
| Faza 1 | Packages visible on `/plans` from real sync |
| Faza 2 | JWT auth + `me/esims`; pytest `test_phase2_dod.py`; staging `scripts/staging-dod-faza2.sh` (+ `CREATE_SANDBOX=1`) |
| Faza 3 | Polygon USDT prepaid credits (ADR-010); staging env `BILLING_ENABLED` + `POLYGON_*`; `scripts/staging-dod-billing.sh` (deposit → verify → ledger → balance → order) |

## Credit-source gates (post–Faza 3)

New prepaid **credit sources** (vouchers, referral, cashback, …) additionally require [ADR 012](../docs/adr/012-billing-extensibility-rules.md):

- [ ] Credit-source ADR **Accepted** with ADR 012 compliance table (all ✅).
- [ ] Architecture tests pass (money-path + source bypass guards).
- [ ] OpenAPI / API docs updated when a public `/api/v1/billing/` endpoint is added.
- [ ] Metrics for grant / redeem / fail paths.
- [ ] Audit trail confirmed (issuer / actor / reason / ledger reference).
- [ ] Docs updated (`ADR_INDEX`, env notes, runbooks as needed).

## Out of scope for Done

- Production deploy (until [ADR 007](../docs/adr/007-staging-only-until-launch.md) superseded).
- Flutter app deliverables (repo not created).
- Perfect UI polish unless specified in the issue.

## Related

- [CI standard](./ci-standard.md)
- [Branching standard](./branching.md)
- [ADR index](../ADR_INDEX.md)
