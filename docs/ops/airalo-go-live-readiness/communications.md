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

- Counts (20 / 100)
- KPI table from [e2e-evidence.md](./e2e-evidence.md)
- Support load / response times
- Go / no-go recommendation for Phase 7

## Guy update (draft ~ early contact)

Use factual, non-overclaiming language until E2E is proven:

```text
- Sandbox API v2 integrated
- Catalog synchronization completed
- Purchase flow implemented
- Payment handled by our platform
- Automatic eSIM provisioning working
- QR delivery implemented
- Installation flow validated
- End-to-end testing in progress with pilot users
- We expect to request production access after completing our 20-user pilot
```

Adjust bullets to match actual evidence before send.

## Production request

Only after [release-decision.md](./release-decision.md) = **GO**. Attach or link this Evidence Pack summary (10-point E2E + pilot-20 KPI).

## Production confirmation

After credentials + smoke:

- Confirm `AIRALO_SANDBOX=false` (or equivalent) only on production with live partner keys
- Record smoke result in [e2e-evidence.md](./e2e-evidence.md) (production validation section / note)
- End Production Freeze; Phase 8 may open under normal PR discipline

## Related

- [README.md](./README.md)
- [release-decision.md](./release-decision.md)
