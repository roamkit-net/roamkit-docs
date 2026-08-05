# Incident runbook

First response for production incidents. Deploy rollback is one tool — not the only one.

Closes readiness condition **C9**.

## Severity (solo ops)

| Level | Meaning | Response |
|-------|---------|----------|
| P0 | Billing wrong / data loss risk / total outage | Page self; start tree immediately; consider rollback within window |
| P1 | Degraded (elevated 5xx, verify failures) | Investigate within 15 min; hypercare if during Gate D |
| P2 | Non-urgent anomaly | Next business window |

## Tree: elevated 500 errors

```text
500 errors / error budget burn
        │
        ▼
Check GET /health/live and /health/ready
        │
        ├── not OK → DB / Redis / container restart; Traefik backends
        │
        └── OK → Check GET /version (expected SHA?)
                │
                ▼
            Check Sentry (new release? spike?)
                │
                ▼
            Check recent deploy / Release Manifest
                │
                ├── bad deploy in rollback window → ./scripts rollback / previous tag
                │
                └── not deploy-related → app logs, Celery, Polygon RPC
                        │
                        ▼
                    Escalate / document; open incident note
```

## Tree: billing verify / ledger drift

```text
Verify failures or reconcile drift alert
        │
        ▼
Confirm BILLING_ENABLED and Polygon env (no sandbox keys on prod)
        │
        ▼
Check DepositRequest statuses + failed_verify metrics
        │
        ▼
Run billing_reconcile_balances (report only — no auto balance fix)
        │
        ├── drift → human review; compensating CreditService entry if needed
        │
        └── RPC / confirmations → wait or switch RPC; do not rewrite ledger rows
```

## Rollback (deploy)

1. Confirm inside **rollback window** (default 30 min after cutover) or accept extended risk.
2. Use production stack `.previous-tag` / `deploy-production` rollback path (see PRODUCTION_PLAN).
3. Re-check `/health/*`, `/version`, billing smoke.
4. Update Release Manifest and freeze status.

## Escalation

Solo operator: document timeline in ops notes; after Hypercare run [production-retrospective.md](./production-retrospective.md) if cutover-related.

## Observability monitors (production)

Uptime Kuma (`/opt/stacks/kuma`, container `kuma` on `proxy` network):

| Id | Check | Probe URL |
|----|-------|-----------|
| 13 | API live | `http://roamkit-api-production:8000/health/live` |
| 14 | API ready | `http://roamkit-api-production:8000/health/ready` |
| 15 | API version | `http://roamkit-api-production:8000/version` |
| 16 | Web origin | `https://roamkit.net/` |
| 17 | Billing config | `http://roamkit-api-production:8000/api/v1/billing/config/` |

Also verify public paths with curl when diagnosing edge/TLS: `https://api.roamkit.net/...` (see Criterion #2 evidence — Kuma→origin SNI/default-cert caveat).

**Sentry:** production DSN in stack `.env`; check project for new release / spike (see tree above).

**Reconcile drift:** Celery beat + ERROR logs + Sentry — no auto balance correction.

## Related

- [Launch Gates](./launch-gates.md)
- [Gate D cutover](./gate-d-cutover.md)
- [Wallet Operations](./wallet-operations.md) (ADR 017 Failure Domains)
- [Billing dashboard](./billing-dashboard.md)
- [SLO targets](./slo.md)
- [Observability Criterion #2 PASS](./releases/1.0.0/evidence/observability-criterion-2.md)
