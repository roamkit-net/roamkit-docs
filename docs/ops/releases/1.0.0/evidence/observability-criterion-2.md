# Evidence — Phase 4 Exit Criterion #2 (Observability) — PASS

| Field | Value |
|-------|-------|
| Criterion | Phase 4 Exit #2 — Observability |
| Verdict | **PASS** → Criterion #2 **YELLOW → GREEN** |
| Closed at (UTC) | 2026-07-26T11:41:24Z |
| Inventory | [observability-criterion-2-inventory.md](./observability-criterion-2-inventory.md) |
| Gate C pack | [observability.md](./observability.md) (unchanged historical C7) |
| Gate C | **GO** (not reopened) |

## Rollback Owner / Facilitator

```text
Facilitator: Engineering (solo operator / agent execution)
Approval: Operator GO for Criterion #2 (2026-07-26)
```

## DoD checklist

| Requirement | Result | Evidence |
|-------------|:------:|----------|
| Sentry ingest / events | ✅ | `capture_message` event_id `0ec031642dd648b0bdcdf94dc16350c7` (Criterion #2 verify); DSN set in prod `.env` |
| All planned Kuma monitors GREEN | ✅ | ids **13–17** heartbeat status=1 / `200 - OK` at 11:41:24Z |
| `billing/config` monitor exists + passes | ✅ | id **17** `http://roamkit-api-production:8000/api/v1/billing/config/` |
| Public endpoint curl verification | ✅ | `api.roamkit.net` live/ready/version/billing/config + `roamkit.net/` all HTTP 200 |
| Monitor placement (retarget) | ✅ | Web **public** `https://roamkit.net/`; API monitors on **docker DNS** (see Lessons) |
| Alerting per runbook | ✅ | Sentry production project (event above); reconcile-drift → ERROR logs + Sentry (Gate C); Kuma UI for uptime (no external push notifier configured — solo ops) |
| Runbook updated | ✅ | [incident-runbook.md](../../../incident-runbook.md) Observability monitors section |
| Capability / checklist docs | ✅ | capability-status + go-live PR3 |

## Uptime Kuma monitors (final)

| Id | Name | URL | ignore_tls | Status |
|----|------|-----|:----------:|:------:|
| 13 | RoamKit API live | `http://roamkit-api-production:8000/health/live` | 0 | UP |
| 14 | RoamKit API ready | `http://roamkit-api-production:8000/health/ready` | 0 | UP |
| 15 | RoamKit API version | `http://roamkit-api-production:8000/version` | 0 | UP |
| 16 | RoamKit Web origin | `https://roamkit.net/` | 0 | UP |
| 17 | RoamKit billing config | `http://roamkit-api-production:8000/api/v1/billing/config/` | 0 | UP |

## Lessons Learned

```text
What worked:
- Adding billing/config as dedicated monitor (catalog price dependency).
- Docker-network HTTP probes to roamkit-api-production are reliable from Kuma (proxy net).
- Sentry capture_message still works on current N (9989fe).

What should improve:
- Public HTTPS probes from Kuma to api.roamkit.net fail: without SNI Traefik serves DEFAULT CERT (self-signed); with ignore_tls, wrong vhost → 404.
- Kuma has no notification channel configured (empty notification table) — optional Discord/email later.

Actions (backlog, not Criterion #2 blockers):
1. Fix Traefik default certificate / ensure SNI for origin checks, or orange-cloud api.roamkit.net, then retarget API monitors to public HTTPS.
2. Add Kuma notification channel for DOWN events.
3. Optional: sync ROAMKIT_* in deploy scripts (from Criterion #4 lessons).
```

## Out of scope (unchanged)

Criterion #3 Billing E2E · Criterion #1 Gate D · Airalo · custom Observability module.
