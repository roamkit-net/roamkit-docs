# Evidence — Gate C observability

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Gate | C |
| Milestone | Observability (C7) |
| Verdict | **GO** (dress-rehearsal uptime; public Hosts at Gate D) |
| Closed at (UTC) | 2026-07-25T21:52:00Z |
| GO by (role / name) | Engineering (solo operator) |
| API SHA | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` |

## Checklist

| Check | Status | Notes |
|-------|:------:|-------|
| `SENTRY_DSN` in production `.env` | ✅ | Set; api/celery/beat recreated |
| Sentry events from production environment | ✅ | `capture_message` event_id `2defa01685024d34b879278dc37adce1` (verify) · `1fc015da977f4e548f77714fbff23d0e` (reconcile-channel verify); `environment=production` |
| Uptime: `/health/live` | ✅ | Uptime Kuma monitor id **13** — `http://roamkit-api-production:8000/health/live` (heartbeat 200) |
| Uptime: `/health/ready` | ✅ | Kuma id **14** — heartbeat 200 |
| Uptime: `/version` | ✅ | Kuma id **15** — heartbeat 200 |
| Web origin | ✅ | Kuma id **16** — `http://roamkit-web-production:3000/` heartbeat 200 |
| Uptime: `GET /api/v1/billing/config/` | ⏳ | **Add at Gate D** — public JSON must stay 200 (catalog price dependency). Staging: `https://api.staging.roamkit.net/api/v1/billing/config/`; production (after cutover): `https://api.roamkit.net/api/v1/billing/config/`. Expect body with `token_symbol` + `display_decimals`. |
| Reconcile-drift notification channel | ✅ | Celery beat `billing-reconcile-balances-daily` (86400s); `billing_reconcile_balances` → `Checked 1 account(s); found 0 drift(s)`; `BalanceDriftDetected` → ERROR logs (`json-file`); ops visibility also via Sentry production project |

## Notes

- No permanent `/sentry-debug/` route added (avoid open error endpoint on production).
- SDK already in image (`config.sentry.init_sentry`); `send_default_pii=False` (stricter than wizard default).
- Dress-rehearsal: Kuma reaches stack on Docker network `proxy`. `DJANGO_ALLOWED_HOSTS` includes `roamkit-api-production` for hostname probes.
- At Gate D: retarget Kuma monitors to `https://api.roamkit.net` / `https://roamkit.net` after Traefik/DNS cutover.
- Catalog prices depend on `billing/config` (ADR-010 graceful degradation). Monitor that endpoint separately from `/health/ready` so a config outage is visible without removing the node from the LB.
- Custom in-house Observability module remains **backlog after Gate D** (not required for this GO).
