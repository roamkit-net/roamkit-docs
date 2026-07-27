# Release decision — Request Airalo Production Access

Complete **before** asking Guy for Airalo production credentials.

## Preconditions checklist

- [x] Wave 1 Acceptance passed ([acceptance.md](./acceptance.md))
- [x] Airalo Sandbox E2E passed ([e2e-evidence.md](./e2e-evidence.md)) — GREEN WITH CONDITIONS on #9 provider ACK
- [x] Evidence Pack complete (readiness + Phase 6 pilot sections filled)
- [x] Production infrastructure ready ([ADR 013](../../adr/013-production-launch.md) / [go-live checklist](../production-go-live-checklist.md)) — Phase 4 GREEN 2026-07-26
- [x] Pilot with 20 users successful (KPI in [e2e-evidence.md](./e2e-evidence.md)) — also Pilot 100 PASSED
- [x] No critical (P0/P1) bugs open *(top-up HTTP 500→502 freeze-allowed fix merged; sandbox top-up provider ACK still rate-limit conditioned from Phase 5)*
- [x] Operational / support runbook verified ([support-runbook.md](./support-runbook.md))
- [x] Support can complete the full user journey *(pilot support samples 5/5 + 21/21; max lookup 1 s)*
- [x] Rollback procedure reviewed ([rollback.md](./rollback.md))

## Parallel waiver (optional)

If Phase 5 Wave 1 / sandbox work ran in parallel with unfinished Phase 4 PR2:

| Field | Value |
|-------|-------|
| Waiver granted? | Yes (historical) — Wave 1 merged to `develop` while Phase 4 Exit Criteria closed; Phase 4 now GREEN so waiver is informational only |
| Scope | Staging Wave 1 + sandbox E2E |
| Granted by | Program plan (parallel waiver clause) |
| Date (UTC) | 2026-07-26 |
| Notes | Phase 4 close-out [phase-4-complete.md](../releases/1.0.0/evidence/phase-4-complete.md); Phase 5 [phase-5-complete.md](../releases/1.0.0/evidence/phase-5-complete.md); Phase 6 [phase-6-complete.md](../releases/1.0.0/evidence/phase-6-complete.md) |

## Decision

| Field | Value |
|-------|-------|
| Verdict | **GO** — Pilot 20 + Pilot 100 KPI met; Production Request DoD green |
| Date (UTC) | 2026-07-27T00:24:00Z |
| Approved by | RoamKit ops (Phase 6 close-out) |
| Next action | Phase 7 — send production request per [communications.md](./communications.md) |

Only on **GO**: proceed to production request per [communications.md](./communications.md).

## Related

- [README.md](./README.md)
- [Release Decision Log](../release-decision-log.md) (platform gates; link this decision when filed)
