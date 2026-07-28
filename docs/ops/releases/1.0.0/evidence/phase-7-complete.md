# Evidence — Phase 7 Complete (Airalo Production Switch)

| Field | Value |
|-------|-------|
| Program phase | Phase 7 — Airalo Production Switch |
| Closed at (UTC) | 2026-07-28T11:32:50Z |
| Verdict | **GREEN WITH CONDITIONS** |
| Condition | Staging sandbox top-up (#9) Airalo HTTP 429 → RoamKit 502 (same class as Phase 5); **production live top-up PASS** |
| Next focus | **Phase 8 — Customer Experience Enhancements**; Production Freeze **ended** |

## Exit criteria

| Criterion | Status | Evidence |
|-----------|:------:|----------|
| Fine Star live Partner API | 🟢 | Guy Live API Partner email; prod `AIRALO_SANDBOX=false` |
| Staging = Roamkit-Sandbox (isolated) | 🟢 | Distinct `client_id`; Fine Star on staging denylist; preflight PASS |
| Phase 7 preflight | 🟢 | `phase7-airalo-preflight.sh` PASS 2026-07-28 |
| Production live smoke (10-point) | 🟢 **10/10** | [api-logs/phase7/](../../airalo-go-live-readiness/api-logs/phase7/) |
| Staging smoke after sandbox keys | 🟡 **9/10** | [api-logs/phase7-staging/](../../airalo-go-live-readiness/api-logs/phase7-staging/) — #9 rate-limit |
| No staging refs to production Airalo credentials | 🟢 | Preflight isolation + blocked client_id |

## Production live smoke

| Field | Value |
|-------|-------|
| Script | `production-dod-airalo-phase7.sh` |
| `CONFIRM_LIVE_ORDER` | `1` |
| API SHA | `407abbda1249652fda5b885c13525c34afaea55d` |
| Package | `discover-in-3days-300mb` |
| Order / eSIM | `7` / `4` |
| External order | `2208690` |
| Top-up | PASS (`discover-in-30days-200mb-topup`) |
| Summary | [summary.json](../../airalo-go-live-readiness/api-logs/phase7/summary.json) |

## Staging smoke (Roamkit-Sandbox)

| Field | Value |
|-------|-------|
| Script | `staging-dod-airalo-phase5.sh` (sandbox E2E) |
| Order / eSIM | `186` / `148` |
| External order | `82901` |
| Pass / Fail | 9 / 1 |
| #9 failure | Airalo sandbox `429 Too Many Requests` (mapped to HTTP 502); compensating refund OK |
| Summary | [summary.json](../../airalo-go-live-readiness/api-logs/phase7-staging/summary.json) |

## Production Freeze

**Ended** 2026-07-28T11:32:50Z. Phase 8 CX and voucher UI may proceed under normal PR discipline.
Allowed during freeze remains historical; new feature work no longer blocked by Phase 7.

## Related

- [production-switch.md](../../airalo-go-live-readiness/production-switch.md)
- [e2e-evidence.md](../../airalo-go-live-readiness/e2e-evidence.md)
- [communications.md](../../airalo-go-live-readiness/communications.md)
- [Phase 6 complete](./phase-6-complete.md)
