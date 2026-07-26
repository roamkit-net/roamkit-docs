# Airalo Go-Live Readiness — Evidence Pack

Single package for **internal review** and **Airalo partner communication**.
Architecture stays in ADRs; this pack is operational proof.

**Related roadmap:** workspace `.cursor/plans/roamkit.plan.md` (Phase 4–8 locked).  
**Not this pack:** Faza 4 Launch Gate audit lives under [`../releases/1.0.0/evidence/`](../releases/1.0.0/evidence/).

## Contents

| Document | Purpose |
|----------|---------|
| [acceptance.md](./acceptance.md) | Wave 1 acceptance checklist (ADR 014 / RFC 002) |
| [e2e-evidence.md](./e2e-evidence.md) | Sandbox E2E (10 points) + pilot KPI results |
| [support-runbook.md](./support-runbook.md) | How support finds an order and walks the user journey |
| [release-decision.md](./release-decision.md) | GO / NO-GO before Airalo production request |
| [rollback.md](./rollback.md) | Feature-flag / deploy rollback + verification |
| [communications.md](./communications.md) | Internal weekly → Guy → production request |
| `screenshots/` | UI / QR / install proof |
| `api-logs/` | Redacted Airalo / API traces |
| `telemetry/` | Install / lifecycle event samples |

## Production Freeze

| | |
|--|--|
| **Starts** | Phase 5 (Airalo Go-Live Readiness) |
| **Ends** | After Phase 7 (Airalo Production Switch) |
| **Allowed** | Bugfix, stability, observability, docs, E2E evidence, performance if blocking pilot |
| **Default blocked** | New billing features, vouchers (ADR 011), subscriptions UI, Phase 8 CX, new providers — everything else needs approval |

## Phase map (Entry / Exit)

| Phase | Name | Entry (summary) | Exit (summary) |
|-------|------|-----------------|----------------|
| 4 | Production Infrastructure Readiness | ADR 013 Accepted | Stack, observability, billing E2E, rollback verified |
| 5 | Airalo Go-Live Readiness | Phase 4 GREEN or parallel waiver; sandbox creds | Wave 1 + sandbox E2E + this pack filled — **GREEN** ([phase-5-complete](../releases/1.0.0/evidence/phase-5-complete.md)) |
| 6 | Pilot Validation (20 → 100) | Phase 5 GREEN; no P0/P1; pack exists | 20 + 100 pilots + KPI met |
| 7 | Airalo Production Switch | Production Request DoD green | Prod credentials + validation smoke |
| 8 | Customer Experience Enhancements | Phase 7 done; freeze ended | Scope reopened |

## Guy gate (sandbox E2E — 10 points)

Proof for partner review (fill in [e2e-evidence.md](./e2e-evidence.md)):

1. Catalog browse
2. Package purchase
3. Charge via RoamKit billing (not Airalo payment)
4. Airalo API provisioning
5. QR generated
6. eSIM installation
7. Activation
8. Data traffic confirmed
9. Top-up works
10. Lifecycle statuses + support can find the order

## Risk register

| Risk | Mitigation |
|------|------------|
| Airalo API change | API contract / provider tests |
| Sandbox ≠ Production | Dedicated production smoke after switch |
| Pilot support overload | [support-runbook.md](./support-runbook.md) + response KPI |
| Billing regression | Architecture tests + reconcile jobs |

## Definition of Success

```text
Roadmap Success =
  Production platform
  + Airalo production
  + 100 successful pilot users
  + No P0/P1 issues
  + Stable billing
  + Stable provisioning
  + Operational support
  + Evidence Pack complete
```

## Related

- [ADR 010](../../adr/010-polygon-usdt-prepaid-credits.md) — billing separated from Airalo
- [ADR 013](../../adr/013-production-launch.md) — platform production
- [ADR 014](../../adr/014-esim-lifecycle-install-telemetry.md) — Wave 1 lifecycle
- [RFC 002](../../rfcs/002-post-purchase-onboarding.md) — Wave 1 onboarding
- [Operations Handbook](../README.md)
- [Production go-live checklist](../production-go-live-checklist.md)
