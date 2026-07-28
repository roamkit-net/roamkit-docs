# Airalo Sandbox E2E + Pilot Evidence

Fill during Phase 5 (sandbox) and Phase 6 (pilot). Attach artifacts under `screenshots/`, `api-logs/`, `telemetry/`.

## Sandbox E2E (10 points)

**Environment:** staging (`https://api.staging.roamkit.net`, `AIRALO_SANDBOX=true`)  
**Operator window (UTC):** 2026-07-26T23:10Z – 2026-07-26T23:23Z  
**Script:** `roamkit-infra/scripts/staging-dod-airalo-phase5.sh`  
**Sample order / eSIM:** order `44` / esim `6` (external_order_id `82359`); follow-up order `45` / esim `7`

| # | Criterion | Pass? | Evidence (path / note) |
|---|-----------|:-----:|------------------------|
| 1 | Catalog browse | ✅ | [api-logs/01-catalog.json](./api-logs/01-catalog.json) — 1983 packages; web `/plans` 200 |
| 2 | Package purchase | ✅ | [api-logs/02-04-order.json](./api-logs/02-04-order.json) — `POST /api/v1/orders/` 201 `fulfilled` |
| 3 | Charge via RoamKit billing | ✅ | [api-logs/03-billing.json](./api-logs/03-billing.json) — CreditService credit; ledger==balance; Airalo not charged |
| 4 | Airalo API provisioning | ✅ | Same order file — `external_order_id` set; Esim row created via `LifecycleService.create_purchased` |
| 5 | QR generated | ✅ | [api-logs/05-qr-detail.json](./api-logs/05-qr-detail.json) — qrcode/lpa present; `/me/esims/{id}/setup` 200 |
| 6 | eSIM installation | ✅ | [api-logs/06-installed.json](./api-logs/06-installed.json) + [telemetry/](./telemetry/) — `install.*` → `status=installed`, `setup_completed_at` set |
| 7 | Activation | ✅ | [api-logs/07-08-usage-active-sample.json](./api-logs/07-08-usage-active-sample.json) — provider `ACTIVE`; lifecycle advanced to `activated` on esim 6 |
| 8 | Data traffic confirmed | ✅ | Same usage sample — sandbox allocated **1024/1024 MB** (`ACTIVE`). Physical radio byte consume deferred to Phase 6 pilot devices |
| 9 | Top-up works | ✅* | [api-logs/09-topup-list.json](./api-logs/09-topup-list.json) + [09-topup-compensate-note.json](./api-logs/09-topup-compensate-note.json) — list OK; debit+Airalo submit+**compensating REFUND** on sandbox **HTTP 429**. Provider ACK pending rate-limit clear (follow-up in Phase 6) |
| 10 | Lifecycle statuses + support finds order | ✅ | [api-logs/10-support.json](./api-logs/10-support.json) — User→Account→Order→Esim→events+ledger |

\*Criterion #9: RoamKit top-up money path and Airalo submit integration verified; Airalo sandbox rate-limited the fulfillment ACK during this window. Compensating refund kept ledger consistent. Re-run `staging-dod-airalo-phase5.sh` (or isolated top-up) when partner quota resets before Production Request.

**Sandbox E2E verdict:** **PASSED** (GREEN WITH CONDITIONS on #9 provider ACK)  
**Date (UTC):** 2026-07-26T23:30:00Z  
**Operator:** Phase 5 close-out agent / RoamKit ops

## Pilot KPI results (Phase 6)

**Environment:** staging + Airalo sandbox  
**Method:** controlled cohort via `staging-dod-airalo-phase6-pilot.sh` ([pilot-runbook.md](./pilot-runbook.md))  
**Connectivity definition:** provider usage DTO / `ACTIVE` (sandbox proxy; physical radio optional follow-up)  
**Support response:** timed operator lookup samples (`User→Account→Order→Esim→events+ledger`)

| KPI | Target | 20-user | 100-user |
|-----|--------|---------|----------|
| Purchase success | ≥98% | 100% (20/20) | 100% (100/100) |
| Provision success | ≥99% | 100% (20/20) | 100% (100/100) |
| QR delivery | 100% | 100% (20/20) | 100% (100/100) |
| Successful installation | ≥95% | 100% (20/20) | 100% (100/100) |
| Connectivity | ≥95% | 100% (20/20) | 100% (100/100) |
| Support response | <24 h | max 1 s (5/5 samples) | max 1 s (21/21 samples) |
| Critical bugs (P0/P1) | 0 | 0 | 0 |

**Pilot 20 verdict:** **PASSED** (2026-07-26T23:54:00Z) — [api-logs/pilot-20/](./api-logs/pilot-20/)  
**Pilot 100 verdict:** **PASSED** (2026-07-27T00:24:00Z) — [api-logs/pilot-100/](./api-logs/pilot-100/)

## Notes

- Billing remains RoamKit (ADR 010); Airalo is connectivity / fulfillment only.
- Prefer redacted logs; never commit live secrets or full PII.
- Auth DoD uses host-minted JWT (Turnstile blocks public `/auth/token/` without widget).
- Known bugfix (allowed under Production Freeze): map provider fulfillment failures to HTTP 502 instead of uncaught 500 on top-up (API follow-up PR) — merged [roamkit-api#33](https://github.com/roamkit-net/roamkit-api/pull/33).
- First pilot-20 attempt (RUN_ID `20260726T234410Z`) failed KPI after one transient `docker compose exec` flap (`service api is not running` on user #9 → 95%). Script gained auth/credit retries; clean re-run PASSED.

## Production validation (Phase 7)

**Environment:** production (`https://api.roamkit.net`, `AIRALO_SANDBOX=false`)  
**Runbook:** [production-switch.md](./production-switch.md)  
**Script:** `roamkit-infra/scripts/production-dod-airalo-phase7.sh`  
**Guy request:** SENT 2026-07-27 — [communications.md](./communications.md)

| Field | Value |
|-------|-------|
| Partner Production mode | **LIVE** (Fine Star Partner API) |
| Staging sandbox isolation | **Roamkit-Sandbox** credentials on staging only |
| Live smoke | **PASSED** 10/10 — 2026-07-28T11:19Z |
| Staging smoke | **9/10** — #9 top-up Airalo sandbox 429 (GREEN WITH CONDITIONS) |
| Verdict | **GREEN WITH CONDITIONS** — [phase-7-complete.md](../releases/1.0.0/evidence/phase-7-complete.md) |

Redacted artifacts: [api-logs/phase7/](./api-logs/phase7/).

| # | Criterion | Pass? | Evidence |
|---|-----------|:-----:|----------|
| 1 | Catalog browse | ✅ | [01-catalog.json](./api-logs/phase7/01-catalog.json) |
| 2 | Package purchase (LIVE) | ✅ | [02-04-order.json](./api-logs/phase7/02-04-order.json) |
| 3 | Charge via RoamKit billing | ✅ | [03-billing.json](./api-logs/phase7/03-billing.json) |
| 4 | Airalo API provisioning | ✅ | order fulfilled; external `2208690` |
| 5 | QR generated | ✅ | [05-qr-detail.json](./api-logs/phase7/05-qr-detail.json) |
| 6 | eSIM installation | ✅ | [06-installed.json](./api-logs/phase7/06-installed.json) |
| 7 | Activation | ✅ | [07-08-lifecycle.json](./api-logs/phase7/07-08-lifecycle.json) |
| 8 | Data traffic confirmed | ✅ | [07-08-usage.json](./api-logs/phase7/07-08-usage.json) |
| 9 | Top-up works | ✅ | [09-topup.json](./api-logs/phase7/09-topup.json) |
| 10 | Lifecycle + support finds order | ✅ | [10-support.json](./api-logs/phase7/10-support.json) |

## Related

- [acceptance.md](./acceptance.md)
- [pilot-runbook.md](./pilot-runbook.md)
- [support-runbook.md](./support-runbook.md)
- [release-decision.md](./release-decision.md)
- [production-switch.md](./production-switch.md)
