# Wave 1 Acceptance

Product-UX slice inside **Phase 5 (Airalo Go-Live Readiness)**.

Normative criteria: [ADR 014](../../adr/014-esim-lifecycle-install-telemetry.md), [RFC 002](../../rfcs/002-post-purchase-onboarding.md).

## Checklist

- [x] Transition + unknown non-downgrade tests green
- [x] Architecture test: only `LifecycleService` writes `Esim.status`
- [x] Idempotent install events + ownership tests
- [x] Migration `unused` → `purchased` applied / verified
- [x] Wizard paths: iOS / Android / Desktop
- [x] Activation-policy snapshot on `Esim` at purchase time
- [x] Install telemetry allowlist events recorded
- [x] OpenAPI coverage for events + new fields
- [x] Post-purchase redirect → `/me/esims/[id]/setup`
- [x] Detail page shows **Continue setup** when incomplete

## Evidence

| Item | Proof |
|------|-------|
| API Wave 1 P1/P2 | [roamkit-api#28](https://github.com/roamkit-net/roamkit-api/pull/28) merged `49607bb` — CI success on `develop` |
| Web Wave 1 P3 | [roamkit-web#27](https://github.com/roamkit-net/roamkit-web/pull/27) merged `e30fd81` (+ follow-ups #48–#52 for install UX) |
| Migrations | `esims/0003_esim_lifecycle_wave1.py`, `catalog/0005_package_activation_policy.py` |
| Staging | `api.staging` SHA includes lifecycle; `GET/POST /me/esims/{id}/events/` live; setup route HTTP 200 |
| Telemetry sample | [telemetry/install-events-sample.json](./telemetry/install-events-sample.json) |

## Sign-off

| Field | Value |
|-------|-------|
| Verdict | **PASSED** |
| Date (UTC) | 2026-07-26T23:30:00Z |
| Signed by | RoamKit ops (Phase 5 close-out) |
| Evidence links | API #28 · Web #27 · [e2e-evidence.md](./e2e-evidence.md) · `api-logs/` · `telemetry/` |

## Related

- [e2e-evidence.md](./e2e-evidence.md)
- [release-decision.md](./release-decision.md)
