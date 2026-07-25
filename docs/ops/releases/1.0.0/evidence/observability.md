# Evidence — Gate C observability

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Gate | C |
| Milestone | Observability (C7) |
| Verdict | **NO-GO / paused** (Stop Criteria) |
| Checked at (UTC) | 2026-07-25T21:40:00Z |
| Checked by | Engineering (solo operator) |

## Checklist

| Check | Status | Notes |
|-------|:------:|-------|
| `SENTRY_DSN` in production `.env` | ⏳ | Not provisioned (absent on staging and org secrets) |
| Sentry events from production environment | ⏳ | Blocked on DSN |
| Uptime: `/health/live` | ⏳ | Public prod Host not live yet; localhost OK |
| Uptime: `/health/ready` | ⏳ | Same |
| Uptime: `/version` | ⏳ | Same |
| Reconcile-drift notification channel | ⏳ | Not configured |

## Stop Criteria

Gate C **paused** here. Do not declare full GO or open Gate D until this pack flips to GO.

## Unblock actions (only these)

1. Create/obtain Sentry project DSN for RoamKit production; set `SENTRY_DSN` in `/opt/stacks/roamkit-production/.env`; recreate api/celery/beat.
2. Confirm a test event with `environment=production`.
3. Configure uptime monitors (post–Gate D public URLs, or interim localhost/`gate-c` Host if operator adds one).
4. Configure billing reconcile-drift alert channel (no auto balance fix).
