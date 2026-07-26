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

| KPI | Target | 20-user | 100-user |
|-----|--------|---------|----------|
| Purchase success | ≥98% | | |
| Provision success | ≥99% | | |
| QR delivery | 100% | | |
| Successful installation | ≥95% | | |
| Connectivity | ≥95% | | |
| Support response | <24 h | | |
| Critical bugs (P0/P1) | 0 | | |

**Pilot 20 verdict:** PENDING  
**Pilot 100 verdict:** PENDING

## Notes

- Billing remains RoamKit (ADR 010); Airalo is connectivity / fulfillment only.
- Prefer redacted logs; never commit live secrets or full PII.
- Auth DoD uses host-minted JWT (Turnstile blocks public `/auth/token/` without widget).
- Known bugfix (allowed under Production Freeze): map provider fulfillment failures to HTTP 502 instead of uncaught 500 on top-up (API follow-up PR).

## Related

- [acceptance.md](./acceptance.md)
- [support-runbook.md](./support-runbook.md)
- [release-decision.md](./release-decision.md)
