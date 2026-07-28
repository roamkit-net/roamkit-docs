# SLO targets

Lightweight targets only — not a full SRE error-budget system.

| Signal | Target | Notes |
|--------|--------|-------|
| API availability | **99.9%** | `/health/live` + origin success over calendar month |
| Billing verification success | **99.99%** | Deposit verify completes without false failure (exclude user error / insufficient confirmations) |

Measure after Gate D using uptime checks and billing metrics / [billing dashboard](./billing-dashboard.md). Revisit at Architecture Decision Review (6 months).

## Related

- [Launch Gates](./launch-gates.md)
- [Billing dashboard](./billing-dashboard.md)
