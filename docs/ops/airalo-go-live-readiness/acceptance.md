# Wave 1 Acceptance

Product-UX slice inside **Phase 5 (Airalo Go-Live Readiness)**.

Normative criteria: [ADR 014](../../adr/014-esim-lifecycle-install-telemetry.md), [RFC 002](../../rfcs/002-post-purchase-onboarding.md).

## Checklist

- [ ] Transition + unknown non-downgrade tests green
- [ ] Architecture test: only `LifecycleService` writes `Esim.status`
- [ ] Idempotent install events + ownership tests
- [ ] Migration `unused` → `purchased` applied / verified
- [ ] Wizard paths: iOS / Android / Desktop
- [ ] Activation-policy snapshot on `Esim` at purchase time
- [ ] Install telemetry allowlist events recorded
- [ ] OpenAPI coverage for events + new fields
- [ ] Post-purchase redirect → `/me/esims/[id]/setup`
- [ ] Detail page shows **Continue setup** when incomplete

## Sign-off

| Field | Value |
|-------|-------|
| Verdict | PENDING |
| Date (UTC) | |
| Signed by | |
| Evidence links | PRs / CI / screenshots/ |

## Related

- [e2e-evidence.md](./e2e-evidence.md)
- [release-decision.md](./release-decision.md)
