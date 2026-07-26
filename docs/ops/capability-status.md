# Capability status (Billing Maturity Matrix)

Non-normative ops view of capability lifecycle. Architecture stays in ADRs.

| Capability | Design | Implemented | Production | Observed | Notes |
|------------|:------:|:-----------:|:----------:|:--------:|-------|
| Ledger / CreditService (ADR 010) | ✅ | ✅ | ⏳ | ⏳ | Staging validated; prod after Gate D |
| Deposits / Polygon verify | ✅ | ✅ | ⏳ | ⏳ | |
| Billing HTTP `/api/v1/billing/` | ✅ | ✅ | ⏳ | ⏳ | |
| WalletConnect deposit UX | ✅ | ✅ | ⏳ | ⏳ | Flag OFF at first cutover |
| Subscriptions service | ✅ | ✅ | ❌ | ❌ | `SUBSCRIPTIONS_ENABLED=false` |
| Credit vouchers (ADR 011) | ✅ | ❌ | ❌ | ❌ | Blocked until **after Phase 7**: [voucher-pr1-kickoff.md](./voucher-pr1-kickoff.md); [Airalo freeze](./airalo-go-live-readiness/README.md) |
| Billing extensibility (ADR 012) | ✅ | ✅ (docs) | n/a | n/a | Constitution, not a runtime feature |
| Production platform (ADR 013 PR1) | ✅ | ✅ | ⏳ | ⏳ | Compose ready; not cut over |
| `GET /version` | ✅ | ✅ | ⏳ | ⏳ | Merged Gate C api; Production after deploy |
| Sentry / uptime | ✅ | ✅ | ⏳ | ⏳ | Sentry SDK + `SENTRY_DSN`; uptime still operator |
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
