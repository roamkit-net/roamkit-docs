# Communications — Airalo go-live

Cadence from readiness through partner production switch.

```text
Internal weekly status
        ↓
   Pilot report
        ↓
    Guy update
        ↓
 Production request
        ↓
Production confirmation
```

## Internal weekly status

- Phase status (4–7): GREEN / YELLOW / RED against Entry/Exit
- Open P0/P1
- Evidence Pack gaps
- Next week focus (execution only — no roadmap churn)

## Pilot report

| Field | Value |
|-------|-------|
| Date (UTC) | 2026-07-27T00:24:00Z |
| Pilot 20 | **PASSED** — 100% on all journey KPIs; support max 1 s |
| Pilot 100 | **PASSED** — 100% on all journey KPIs; support max 1 s (21 samples) |
| Support load | Operator lookup drills only (scripted cohort); no P0/P1 |
| Recommendation | **GO** for Phase 7 Production Request |

KPI detail: [e2e-evidence.md](./e2e-evidence.md) · Close-out: [phase-6-complete.md](../releases/1.0.0/evidence/phase-6-complete.md)

## Guy update (draft ~ early contact)

Use factual, non-overclaiming language until E2E is proven:

```text
- Sandbox API v2 integrated
- Catalog synchronization completed
- Purchase flow implemented
- Payment handled by our platform
- Automatic eSIM provisioning working
- QR delivery implemented
- Installation flow validated (guided setup wizard + install telemetry)
- Sandbox E2E 10-point gate passed (top-up provider ACK subject to sandbox rate limits; RoamKit compensating refund verified)
- Controlled pilot completed: 20 users + 100 users — all locked Pilot KPIs met
- We are ready to request production access
```

Adjust bullets to match actual evidence before send.

## Production request

Only after [release-decision.md](./release-decision.md) = **GO**. Attach or link this Evidence Pack summary (10-point E2E + pilot-20 KPI).

| Field | Value |
|-------|-------|
| Status | **SENT** |
| Sent at (UTC) | 2026-07-27T00:41:29Z |
| From | `info@roamkit.net` (RoamKit ops) |
| To | `guy.dor@airalo.com` |
| CC | `avrcan@finestar.hr` |
| Subject | RoamKit / Fine Star — request Airalo Partner API Production switch |
| Message-ID | `<178511288945.1251.6796807227239248739@roamkit.net>` |
| Runbook | [production-switch.md](./production-switch.md) |

### Body (sent)

```text
Hi Guy,

We are ready to request Production (Live) mode for Fine Star d.o.o. / RoamKit
(Partner Application Id 17558).

Readiness summary:
- Sandbox API v2 integrated (catalog sync, orders, top-ups, usage)
- Purchase + payment handled on our platform (Polygon USDT prepaid credits)
- Automatic eSIM provisioning + QR delivery working
- Guided install / lifecycle telemetry validated
- Sandbox E2E 10-point gate passed
- Controlled pilot completed: 20 + 100 users — all locked Pilot KPIs met
- Production platform live at https://roamkit.net and https://api.roamkit.net

Evidence Pack (internal): airalo-go-live-readiness / release-decision = GO

Please:
1) Switch our Partner API account to Production mode
2) Issue a separate Sandbox credential pair for our staging environment
   (staging currently shares the same client_id as production; we need sandbox
   isolation after the switch)
3) Confirm when Production mode is active so we can run one live validation order

Happy to complete any remaining Go Live checklist items in the Partner Platform.

Best regards,
Ante Vrcan
Fine Star d.o.o. / RoamKit
avrcan@finestar.hr · https://roamkit.net
```

## Guy staging demo

Temporary staging walkthrough account for Guy while Production mode is pending.
Password was sent by email only — **never** commit it.

| Field | Value |
|-------|-------|
| Status | **SENT** |
| Sent at (UTC) | 2026-07-27T07:24:53Z |
| From | `info@roamkit.net` |
| To | `guy.dor@airalo.com` |
| CC | `avrcan@finestar.hr` |
| Subject | RoamKit staging demo access — Fine Star / Application 17558 |
| Message-ID | `<178513709373.6177.9142966495017160542@roamkit.net>` |
| Staging URL | `https://staging.roamkit.net/login` |
| Login email | `guy.dor@airalo.com` |
| Credits | `100.000000` via `CreditService.admin_adjust` (`phase7-guy-demo-credit-100`) |
| Caveat | Staging remains Airalo **sandbox** (test eSIMs) |

## Guy checklist confirmation

Threaded reply to Guy’s Production checklist (honesty lock — no overclaim; v1 webhooks not enabled).
Phase 7 remains **blocked** until Guy confirms Production switch + live smoke.

| Field | Value |
|-------|-------|
| Status | **SENT** |
| Sent at (UTC) | 2026-07-27T13:26:33Z |
| From | `info@roamkit.net` |
| To | `guy.dor@airalo.com` |
| CC | `avrcan@finestar.hr` |
| Subject | `Re: RoamKit staging demo access — Fine Star / Application 17558` |
| Message-ID | `<178515879474.10513.11020445882123476370@roamkit.net>` |
| In-Reply-To | `<CAAqmYC5i5JeJ5=6nr2FJF5w6RSWT79OfmEyT00jM0vwJyf20yQ@mail.gmail.com>` |
| OpenAPI | `https://api.staging.roamkit.net/api/docs/` · `/api/redoc/` · `/api/schema/` |
| Live-order ack | Explicit: after 17558 Production switch, orders/top-ups are real |
| Switch notify | Asked Guy to ping when switch is done → then run production validation smoke |
| Sandbox app (planned) | Email `staging@roamkit.net` · Company `Roamkit-Sandbox` · keep **permanently** in Sandbox |
| Sandbox registration | Pending human Partner Platform signup; will notify Guy when submitted |
| Honesty notes | Edge cases = go-live scope only; Airalo Partner webhooks **not** enabled for v1 |

## Production confirmation

After credentials + smoke ([production-switch.md](./production-switch.md)):

- Confirm `AIRALO_SANDBOX=false` only on production; staging uses dedicated sandbox keys
- Record smoke result in [e2e-evidence.md](./e2e-evidence.md) (production validation section)
- End Production Freeze; Phase 8 may open under normal PR discipline

| Field | Value |
|-------|-------|
| Status | PENDING Guy confirmation + live smoke |
| Production env | `AIRALO_SANDBOX` still `true` until switch confirmed |
| Smoke | Not run (awaits Production mode) |

## Related

- [README.md](./README.md)
- [release-decision.md](./release-decision.md)
- [production-switch.md](./production-switch.md)
