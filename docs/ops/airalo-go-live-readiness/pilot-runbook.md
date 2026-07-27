# Pilot runbook — Phase 6

Controlled pilot on **staging + Airalo sandbox** measuring locked Pilot KPIs
before requesting Airalo production access.

## Scope

| Cohort | Purpose | Exit |
|--------|---------|------|
| Pilot 20 | First gate for Production Request DoD | All KPI targets met |
| Pilot 100 | Scale confidence before / with Phase 7 | All KPI targets met |

**Environment:** `https://api.staging.roamkit.net` (`AIRALO_SANDBOX=true`)  
**Billing:** RoamKit CreditService stand-in deposits (ADR 010) — Airalo is fulfillment only.  
**Out of band:** Physical device radio byte consume may be sampled separately; connectivity KPI uses provider usage DTO / `ACTIVE` as the sandbox proxy (same as Phase 5 criterion #8).

## KPI targets (locked)

| KPI | Target |
|-----|--------|
| Purchase success | ≥98% |
| Provision success | ≥99% |
| QR delivery | 100% |
| Successful installation | ≥95% |
| Connectivity | ≥95% |
| Support response | <24 h |
| Critical bugs (P0/P1) | 0 |

## How to run

On the staging host:

```bash
cd /opt/stacks/roamkit-net

# Pilot 20
mkdir -p /tmp/phase6-pilot-20
COHORT_SIZE=20 USER_DELAY_SEC=8 EVIDENCE_DIR=/tmp/phase6-pilot-20 \
  ./scripts/staging-dod-airalo-phase6-pilot.sh

# Pilot 100 (after pilot 20 GREEN; raise delay if Airalo rate-limits)
mkdir -p /tmp/phase6-pilot-100
COHORT_SIZE=100 USER_DELAY_SEC=10 EVIDENCE_DIR=/tmp/phase6-pilot-100 \
  ./scripts/staging-dod-airalo-phase6-pilot.sh
```

Script source: `roamkit-infra/scripts/staging-dod-airalo-phase6-pilot.sh`.

### What each user journey covers

1. Host-minted JWT (Turnstile bypass for ops)
2. CreditService deposit stand-in
3. Package purchase → Airalo provision
4. QR / LPA on My eSIM
5. Install telemetry funnel → `installed`
6. Usage sync (connectivity proxy)
7. Sampled support trail lookup (timed)

Top-up is **not** repeated per pilot user (Phase 5 covered money path + compensating refund under sandbox 429).

## Recording results

1. Copy `kpi-summary.json` + `results.tsv` (+ sample redacted orders) into Evidence Pack:
   - `docs/ops/airalo-go-live-readiness/api-logs/pilot-20/`
   - `docs/ops/airalo-go-live-readiness/api-logs/pilot-100/`
2. Fill the KPI table in [e2e-evidence.md](./e2e-evidence.md).
3. Update [release-decision.md](./release-decision.md) when pilot 20 is GREEN.
4. Write [phase-6-complete.md](../releases/1.0.0/evidence/phase-6-complete.md) when both cohorts pass.

## Support response drill

Pilot KPI requires first meaningful support response **&lt; 24 h**. The cohort script times operator-style `User → Account → Order → Esim → events + ledger` lookups on sampled users; record max latency in `kpi-summary.json` (`support_response_max_ms`).

Manual drill (optional):

1. Pick a pilot email from `results.tsv`.
2. Walk [support-runbook.md](./support-runbook.md).
3. Note time-to-first-meaningful-response in the pilot report.

## Abort / rate limits

- Airalo HTTP 429 / 5xx on order → script retries with backoff, then counts the user as purchase fail if exhausted.
- Raise `USER_DELAY_SEC` and resume with a fresh `RUN_ID` (new emails); do not re-use failed idempotency keys across runs when interpreting KPIs — prefer a clean cohort.
- P0/P1 discovered mid-pilot → **stop**, fix under Production Freeze, restart cohort.

## Related

- [e2e-evidence.md](./e2e-evidence.md)
- [support-runbook.md](./support-runbook.md)
- [communications.md](./communications.md) — Pilot report section
- [release-decision.md](./release-decision.md)
