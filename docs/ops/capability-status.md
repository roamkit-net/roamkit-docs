# Capability status (Implemented vs Production Enabled)

Non-normative ops view of what exists in code versus what is live for customers.
Architecture decisions remain in ADRs; this table tracks **delivery state**.

| Capability | Implemented | Production Enabled | Notes |
|------------|:-----------:|:------------------:|-------|
| Polygon USDT prepaid billing (ADR 010) | ✅ | ❌ | Staging validated; waiting Faza 4 PR2 cutover |
| Billing HTTP `/api/v1/billing/` | ✅ | ❌ | Enabled on staging; prod after cutover |
| WalletConnect deposit UX | ✅ | ❌ | Flag OFF at first cutover (ADR 013) |
| Subscriptions service | ✅ | ❌ | Code present; `SUBSCRIPTIONS_ENABLED=false` |
| Credit vouchers (ADR 011) | ❌ | ❌ | ADR Accepted; voucher PR 1–3 not started |
| Billing extensibility rules (ADR 012) | ✅ (docs) | n/a | Normative constitution; not a runtime feature |
| Production platform stack (ADR 013 PR1) | ✅ | ❌ | Compose/bootstrap ready; stack not cut over |
| `GET /version` | ❌ | ❌ | Contract documented in PRODUCTION_PLAN; api PR2 |
| Sentry / uptime alerts | ❌ | ❌ | Faza 4 PR3 |

**Definitions**

- **Implemented** — merged to `develop` (and/or documented ADR) and usable in staging or as design.
- **Production Enabled** — running on `/opt/stacks/roamkit-production/` with customer traffic (or intentional prod flag ON).

Update this table at each Faza 4 milestone and when vouchers ship.

## Related

- [ADR 013](../adr/013-production-launch.md)
- [Production readiness review](./production-readiness-review.md)
- [Go-live checklist](./production-go-live-checklist.md)
- Infra runbook: `roamkit-infra/bootstrap/hetzner/PRODUCTION_PLAN.md`
