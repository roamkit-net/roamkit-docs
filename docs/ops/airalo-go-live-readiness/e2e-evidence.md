# Airalo Sandbox E2E + Pilot Evidence

Fill during Phase 5 (sandbox) and Phase 6 (pilot). Attach artifacts under `screenshots/`, `api-logs/`, `telemetry/`.

## Sandbox E2E (10 points)

| # | Criterion | Pass? | Evidence (path / note) |
|---|-----------|:-----:|------------------------|
| 1 | Catalog browse | ⬜ | |
| 2 | Package purchase | ⬜ | |
| 3 | Charge via RoamKit billing | ⬜ | |
| 4 | Airalo API provisioning | ⬜ | |
| 5 | QR generated | ⬜ | |
| 6 | eSIM installation | ⬜ | |
| 7 | Activation | ⬜ | |
| 8 | Data traffic confirmed | ⬜ | |
| 9 | Top-up works | ⬜ | |
| 10 | Lifecycle statuses + support finds order | ⬜ | |

**Sandbox E2E verdict:** PENDING  
**Date (UTC):**  
**Operator:**

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

## Related

- [acceptance.md](./acceptance.md)
- [support-runbook.md](./support-runbook.md)
- [release-decision.md](./release-decision.md)
