# Operations Handbook

Single place for **preparing and running releases**. Architecture decisions stay in
[ADRs](../adr/); this handbook is Release Engineering.

**Program shift:** Architecture (ADR 010–013) is design-frozen. Work here is how we
ship safely — Launch Gates, cutover discipline, incidents, and maturity tracking.

## Contents

| Document | Purpose |
|----------|---------|
| [Launch Gates](./launch-gates.md) | Gate A–D, entry/exit criteria, owners, time-boxes, freeze |
| [Production readiness review](./production-readiness-review.md) | PR0.5 go/no-go scorecard |
| [Go-live checklist](./production-go-live-checklist.md) | Faza 4 cutover checklist / must-haves |
| [Capability status / Maturity Matrix](./capability-status.md) | Design → Implemented → Production → Observed |
| [Release Manifest](./release-manifest.md) | Per-release template; filled copies in [`releases/`](./releases/) |
| [Evidence of Gate](./evidence-of-gate.md) | Audit pack per closed Gate (SHA, CI, GO, PRs) |
| [Migration Ready](./migration-ready.md) | Pre-migrate gate (duration, locks, backup, rollback) |
| [Incident runbook](./incident-runbook.md) | First response trees (not only deploy rollback) |
| [SLO targets](./slo.md) | Lightweight availability / billing targets |
| [Production retrospective](./production-retrospective.md) | Post-hypercare lessons learned template |
| [Billing dashboard](./billing-dashboard.md) | Operator metrics after cutover |
| [Disaster Day](./disaster-day.md) | Annual failure simulation |
| [Gate C exit](./gate-c-exit.md) | Engineering tracker for C* before GO |
| [Gate D cutover](./gate-d-cutover.md) | Time-boxed cutover runbook |
| [Voucher PR1 kickoff](./voucher-pr1-kickoff.md) | Blocked until Gate D + retrospective |

## Related

- [ADR 013 — Production launch](../adr/013-production-launch.md)
- [ADR_INDEX](../../ADR_INDEX.md)
- Infra: `roamkit-infra/bootstrap/hetzner/PRODUCTION_PLAN.md`
