# Capability status (Billing Maturity Matrix)

Non-normative ops view of capability lifecycle. Architecture stays in ADRs.

| Capability | Design | Implemented | Production | Observed | Notes |
|------------|:------:|:-----------:|:----------:|:--------:|-------|
| Ledger / CreditService (ADR 010) | ✅ | ✅ | ✅ | ✅ | Public Billing DoD PASS — Criterion #3 |
| Deposits / Polygon verify | ✅ | ✅ | ✅ | ⏳ | Env+negative path OK; live TX verify optional backlog |
| Billing HTTP `/api/v1/billing/` | ✅ | ✅ | ✅ | ✅ | Criterion #3 public DoD |
| WalletConnect deposit UX | ✅ | ✅ | ⏳ | ⏳ | Flag OFF at first cutover |
| Subscriptions service | ✅ | ✅ | ❌ | ❌ | `SUBSCRIPTIONS_ENABLED=false` |
| Credit vouchers (ADR 011) | ✅ | ❌ | ❌ | ❌ | Blocked until **after Phase 7**: [voucher-pr1-kickoff.md](./voucher-pr1-kickoff.md); [Airalo freeze](./airalo-go-live-readiness/README.md) |
| Billing extensibility (ADR 012) | ✅ | ✅ (docs) | n/a | n/a | Constitution, not a runtime feature |
| Production platform (ADR 013 PR1) | ✅ | ✅ | ✅ | ✅ | Gate D GO + Phase 4 Complete — [phase-4-complete.md](./releases/1.0.0/evidence/phase-4-complete.md) |
| `GET /version` | ✅ | ✅ | ✅ | ✅ | Verified on prod (Criteria #4 / #2) |
| Sentry / uptime | ✅ | ✅ | ✅ | ✅ | Criterion #2 PASS — [observability-criterion-2.md](./releases/1.0.0/evidence/observability-criterion-2.md); Kuma 13–17 |
| OpenAPI / schema (C10) | ✅ | ✅ | ⏳ | n/a | Staging schema/docs; prod after freeze policy |
| Billing dashboard | ✅ | ❌ | ❌ | ❌ | [billing-dashboard.md](./billing-dashboard.md) |

**Column definitions**

- **Design** — ADR Accepted (or explicit N/A design note).
- **Implemented** — merged to `develop`, usable on staging (or docs-only constitution).
- **Production** — running on `/opt/stacks/roamkit-production/` with intended flags (or customer traffic).
- **Observed** — confirmed via metrics / [billing dashboard](./billing-dashboard.md) in production (not only flag ON).

Update at each Launch Gate milestone, Airalo go-live phase exit, and when vouchers ship.

## Related

- [Operations Handbook](./README.md)
- [Launch Gates](./launch-gates.md)
- [Airalo Go-Live Readiness](./airalo-go-live-readiness/README.md)
- [ADR 013](../adr/013-production-launch.md)
- [Production readiness review](./production-readiness-review.md)
- [Go-live checklist](./production-go-live-checklist.md)
