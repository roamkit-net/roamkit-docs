# Release decision — Request Airalo Production Access

Complete **before** asking Guy for Airalo production credentials.

## Preconditions checklist

- [ ] Wave 1 Acceptance passed ([acceptance.md](./acceptance.md))
- [ ] Airalo Sandbox E2E passed ([e2e-evidence.md](./e2e-evidence.md))
- [ ] Evidence Pack complete (this directory filled for readiness + pilot-20 minimum)
- [ ] Production infrastructure ready ([ADR 013](../../adr/013-production-launch.md) / [go-live checklist](../production-go-live-checklist.md))
- [ ] Pilot with 20 users successful (KPI in [e2e-evidence.md](./e2e-evidence.md))
- [ ] No critical (P0/P1) bugs open
- [ ] Operational / support runbook verified ([support-runbook.md](./support-runbook.md))
- [ ] Support can complete the full user journey
- [ ] Rollback procedure reviewed ([rollback.md](./rollback.md))

## Parallel waiver (optional)

If Phase 5 Wave 1 / sandbox work ran in parallel with unfinished Phase 4 PR2:

| Field | Value |
|-------|-------|
| Waiver granted? | No / Yes |
| Scope | e.g. staging-only Wave 1 + sandbox E2E |
| Granted by | |
| Date (UTC) | |
| Notes | |

## Decision

| Field | Value |
|-------|-------|
| Verdict | PENDING — GO \| NO-GO |
| Date (UTC) | |
| Approved by | |
| Next action | Request Airalo Production Access \| Remediate gaps |

Only on **GO**: proceed to production request per [communications.md](./communications.md).

## Related

- [README.md](./README.md)
- [Release Decision Log](../release-decision-log.md) (platform gates; link this decision when filed)
