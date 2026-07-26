# Evidence — Phase 4 Exit Criterion #2 (Observability) — inventory

| Field | Value |
|-------|-------|
| Criterion | Phase 4 Exit #2 — Observability |
| Inventory at (UTC) | 2026-07-26T11:39:00Z |
| Prior Gate C pack | [observability.md](./observability.md) (C7 GO; dress-rehearsal; `billing/config` deferred) |
| Criterion #4 | GREEN (closed) — do not reopen |

## Inventory (DoD map)

| # | Requirement | Status | Notes |
|---|-------------|:------:|-------|
| 1 | Sentry DSN in production `.env` | 🟢 GREEN | Present |
| 2 | Sentry ingest / events (`environment=production`) | 🟢 GREEN | Gate C events recorded; re-verify during closure |
| 3 | Kuma `/health/live` | 🔴 RED | Monitor **13** URL public ✅ but heartbeat **DOWN** (`self-signed certificate`) |
| 4 | Kuma `/health/ready` | 🔴 RED | Monitor **14** — same TLS failure |
| 5 | Kuma `/version` | 🔴 RED | Monitor **15** — same TLS failure |
| 6 | Kuma web origin | 🟢 GREEN | Monitor **16** `https://roamkit.net/` → 200 OK |
| 7 | Kuma `GET /api/v1/billing/config/` | 🔴 RED | **Missing** (deferred at Gate C “Add at Gate D”) |
| 8 | Public retarget of monitors | 🟡 YELLOW | URLs already `api.roamkit.net` / `roamkit.net`; API monitors not healthy |
| 9 | Public endpoints healthy (curl) | 🟢 GREEN | live/ready/version/billing/config/web all HTTP 200 |
| 10 | Reconcile-drift → ops visibility | 🟢 GREEN | Celery beat + ERROR logs + Sentry (Gate C) |
| 11 | Alerting (uptime + Sentry + drift) | 🟡 YELLOW | Sentry path OK; Kuma `notification` table **empty** (no push channel) |
| 12 | Incident runbook | 🟢 GREEN | [incident-runbook.md](../../../incident-runbook.md) present |
| 13 | Capability-status / checklist docs | 🟡 YELLOW | Still shows Sentry/uptime Production ⏳; PR3 boxes unchecked |

**Criterion #2 overall:** 🟢 **GREEN** (closed 2026-07-26 — [observability-criterion-2.md](./observability-criterion-2.md)).

### Post-closure status

| # | Requirement | Final |
|---|-------------|:-----:|
| 1–2 | Sentry | 🟢 |
| 3–6 | Kuma live/ready/version/web | 🟢 |
| 7 | billing/config monitor | 🟢 id 17 |
| 8 | Monitor placement | 🟢 hybrid (API docker DNS; web public) |
| 9 | Public curl | 🟢 |
| 10 | Reconcile drift | 🟢 |
| 11 | Alerting | 🟢 Sentry + Kuma UI (push notifier backlog) |
| 12–13 | Runbook + docs | 🟢 |

## Root cause (API Kuma DOWN)

Connecting to `api.roamkit.net:443` **without SNI** yields Traefik **DEFAULT CERT** (self-signed). With SNI → Let's Encrypt `api.roamkit.net`. Uptime Kuma heartbeats fail with `self-signed certificate` while host/`node` HTTPS with SNI succeeds.

`roamkit.net` (Cloudflare) monitor stays UP.

## Closure plan (this step only)

1. Fix monitors 13–15 to GREEN (prefer public URL + correct TLS behaviour; fallback `ignore_tls` or docker DNS HTTP if needed).
2. Add monitor for `https://api.roamkit.net/api/v1/billing/config/` (expect 200 + JSON fields).
3. Re-verify Sentry event.
4. Confirm alerting stance (Sentry + drift; document Kuma notification gap or add channel).
5. Update capability-status + go-live PR3 + Criterion #2 evidence → GREEN if DoD met.

## Out of scope

Billing E2E (#3), Gate D (#1), Airalo, custom Observability module backlog.
