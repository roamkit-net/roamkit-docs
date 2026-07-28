# Billing dashboard

First screen the operator watches after Gate D. Build after Hypercare / retrospective;
until then use `billing_metrics` management command + Sentry/uptime.

## Required panels

| Panel | Source / idea |
|-------|----------------|
| Deposit count | `BillingMetrics.deposit_count` |
| Verification success rate | completed / (completed + failed) deposits |
| Ledger drift | reconcile job / drift events (alert, no auto-fix) |
| Reconcile status | last run time + drift count |
| Refund count | ledger `REFUND` / compensating entries |
| CreditService latency | APM / Sentry performance (p95 spend/credit) |

## Maturity

When panels are live and green for Hypercare+, set **Observed** ✅ on Ledger / Deposits in
[capability-status.md](./capability-status.md).

## Related

- [SLO targets](./slo.md)
- [Incident runbook](./incident-runbook.md)
- API: `apps.billing.services.metrics.collect_billing_metrics`
