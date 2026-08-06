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
| [Release Decision Log](./release-decision-log.md) | Chronological GO / NO-GO index |
| [Migration Ready](./migration-ready.md) | Pre-migrate gate (duration, locks, backup, rollback) |
| [Incident runbook](./incident-runbook.md) | First response trees (not only deploy rollback) |
| [Wallet Operations](./wallet-operations.md) | ADR 017 Failure Domains recovery + wallet drills |
| [Wallet Product Activation](./wallet-product-activation.md) | ADR 018 cutover phases, flags, readiness, rollback |
| [Phase 2 Validation Report](./wallet-phase-2-validation.md) | Limited Traffic KPI gate before PR5 / Phase 3 |
| [Pricing Validation Checklist](./pricing-validation-checklist.md) | ADR 019 staging gate before pricing API / web (PR4+) |
| [Pricing Validation Report](./pricing-validation-report.md) | Official PASS/FAIL decision record after staging (unlocks PR4) |
| [SLO targets](./slo.md) | Lightweight availability / billing targets |
| [Production retrospective](./production-retrospective.md) | Post-hypercare lessons learned template |
| [Billing dashboard](./billing-dashboard.md) | Operator metrics after cutover |
| [Disaster Day](./disaster-day.md) | Annual failure simulation |
| [Rollback candidate qualification](./rollback-candidate-qualification.md) | N/N−1 must share migration **and** smoke/health contract |
| [Gate C exit](./gate-c-exit.md) | Engineering tracker for C* before GO |
| [Gate D cutover](./gate-d-cutover.md) | Time-boxed cutover runbook |
| [Airalo Go-Live Readiness](./airalo-go-live-readiness/README.md) | Partner Evidence Pack (Phase 5–7); Production Freeze |
| [Airalo removable eUICC (future)](./airalo-removable-euicc.md) | Partner status: on hold; what we asked / what to ask later |
| [Cloudflare auth protection](./cloudflare-auth-protection.md) | Turnstile provisioning, soak, Managed Challenge, metrics |
| [Google OAuth](./google-oauth.md) | GIS ID-token clients, enable order, support, metrics |
| [Android LPA deep link spike](./android-lpa-deep-link-spike.md) | Matrix + Decision Log for Android `LPA:` install CTA |
| [Voucher PR1 kickoff](./voucher-pr1-kickoff.md) | Blocked until after Phase 7 Airalo production switch |

## Related

- [ADR 013 — Production launch](../adr/013-production-launch.md)
- [ADR_INDEX](../../ADR_INDEX.md)
- Infra: `roamkit-infra/bootstrap/hetzner/PRODUCTION_PLAN.md`
