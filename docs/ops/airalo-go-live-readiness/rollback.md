# Rollback — Airalo / production readiness

Use if a pilot or production validation fails critically. Prefer the smallest safe stop.

## Procedure

1. **Disable feature flag(s)** relevant to the failing path (e.g. pause purchases / billing-facing flags per ADR 013 matrix — do not invent new flags here).
2. **Revert deployment** to last known-good image tag (`.previous-tag` / deploy rollback script on host).
3. **Verify billing** — balances, recent ledger entries, no stuck deposits; run billing smoke if available.
4. **Verify orders** — no half-provisioned paid orders without compensating refund path.
5. **Verify provisioning** — Airalo/sandbox outstanding orders; document any manual partner follow-up.
6. **Smoke tests** — health/`/version`, catalog, auth, one dry billing check as appropriate to environment.

## After rollback

- [ ] Incident note (link [incident-runbook.md](../incident-runbook.md))
- [ ] P0/P1 filed; Phase 7 blocked until clear
- [ ] Update [e2e-evidence.md](./e2e-evidence.md) / [release-decision.md](./release-decision.md) to NO-GO if needed
- [ ] Communicate status per [communications.md](./communications.md)

## Related

- [Gate D cutover](../gate-d-cutover.md)
- Infra: `roamkit-infra/bootstrap/hetzner/PRODUCTION_PLAN.md`
