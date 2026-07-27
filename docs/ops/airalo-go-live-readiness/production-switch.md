# Phase 7 — Airalo Production Switch

Partner-side cutover from Sandbox → Production for Fine Star / RoamKit.
Same API credentials and base URL (`https://partners-api.airalo.com`); mode is
controlled by Airalo on the partner account ([Sandbox mode docs](https://developers.partners.airalo.com/sandbox-mode-2114305m0)).

## Preconditions

- [release-decision.md](./release-decision.md) = **GO**
- Phase 6 Pilot 20 (+ Pilot 100) GREEN
- Production Request DoD checklist complete
- Staging and production use **separate** Airalo partner applications:
  - staging → **Roamkit-Sandbox**
  - production → **Fine Star** (Application Id **17558**)
- Staging Traefik / ALLOWED_HOSTS serve **only** `staging.roamkit.net` (no apex)

## Sequence

```text
1. Send Production request to Guy (communications.md)
2. Complete Partner Platform Go Live checklist
3. Guy / Airalo approves Fine Star live + Roamkit-Sandbox for staging
4. Pause staging Airalo (fail closed) until sandbox keys installed
5. Staging .env: Roamkit-Sandbox keys, AIRALO_SANDBOX=true, AIRALO_ENABLED=true
   + AIRALO_BLOCKED_CLIENT_IDS=<Fine Star live client_id>
6. Production .env: Fine Star live keys, AIRALO_SANDBOX=false, AIRALO_ENABLED=true
7. Partner identity verify + phase7-airalo-preflight.sh (STOP if any fail)
8. CONFIRM_LIVE_ORDER=1 ./scripts/production-dod-airalo-phase7.sh
9. Staging smoke PASS after sandbox credentials
10. Record evidence → Phase 7 GREEN → end Production Freeze
```

## Production request

Draft / sent status: [communications.md](./communications.md).

To: `guy.dor@airalo.com` · From ops mailbox `info@roamkit.net` · CC operator.

Ask for:

1. Switch Fine Star (Application Id **17558**) Partner API account to **Production**
2. Separate **sandbox** credentials for RoamKit staging (`api.staging.roamkit.net`)
3. Confirmation when Production mode is active

## Staging pause (before Partner Production flip)

Staging must never share Fine Star live credentials. **Before** Guy flips
Production mode (and before deploying Airalo startup guards with empty creds):

```bash
cd /opt/stacks/roamkit-net
./scripts/staging-pause-airalo.sh
# sets AIRALO_CLIENT_ID/SECRET empty, AIRALO_SANDBOX=true, AIRALO_ENABLED=false
```

After dedicated **Roamkit-Sandbox** credentials arrive:

```bash
# Set sandbox AIRALO_CLIENT_ID / AIRALO_CLIENT_SECRET in .env
# AIRALO_SANDBOX=true
# AIRALO_ENABLED=true
# AIRALO_BLOCKED_CLIENT_IDS=<Fine Star live client_id>
docker compose --profile app up -d --force-recreate api celery celery-beat
```

Verify:

```text
staging partner account == Roamkit-Sandbox
production partner account == Fine Star
No staging configuration references production Airalo credentials.
```

## Flip production env

On `/opt/stacks/roamkit-production/`:

```bash
# After Guy confirms Production mode is active:
# AIRALO_SANDBOX=false
# AIRALO_ENABLED=true
# Fine Star live AIRALO_CLIENT_ID / AIRALO_CLIENT_SECRET
grep '^AIRALO_' .env | sed -E 's/(CLIENT_ID|CLIENT_SECRET)=.*/\1=***REDACTED***/'
docker compose --profile app up -d --force-recreate api celery celery-beat
docker compose --profile app exec -T api python -c \
  'from django.conf import settings; assert settings.AIRALO_SANDBOX is False; assert settings.AIRALO_ENABLED is True; print("OK")'
```

## Preflight (mandatory before live smoke)

Hard **STOP** if any check fails. Script:
`roamkit-infra/scripts/phase7-airalo-preflight.sh`.

```bash
cd /opt/stacks/roamkit-production
EXPECTED_GIT_SHA=<prod api git_sha> \
CONFIRM_SENTRY_GREEN=1 CONFIRM_KUMA_GREEN=1 \
CONFIRM_STAGING_PARTNER=Roamkit-Sandbox \
CONFIRM_PRODUCTION_PARTNER='Fine Star' \
  ./scripts/phase7-airalo-preflight.sh
```

Checks:

- production `AIRALO_SANDBOX=false`, `AIRALO_ENABLED=true`, credentials present
- staging cannot complete an Airalo request (disabled / empty / blocked)
- staging `client_id` ≠ production Fine Star `client_id`
- production `GET /version` matches `EXPECTED_GIT_SHA`
- operator attestation: Sentry GREEN, Kuma GREEN, partner identities

Startup guards only check **config consistency** (flags + presence). Credential
**validity** is validated here and by live/staging smoke — not at Django boot.

## Production validation smoke

```bash
cd /opt/stacks/roamkit-production
mkdir -p /tmp/phase7-evidence
CONFIRM_LIVE_ORDER=1 INCLUDE_TOPUP=1 EVIDENCE_DIR=/tmp/phase7-evidence \
  ./scripts/production-dod-airalo-phase7.sh
```

Script: `roamkit-infra/scripts/production-dod-airalo-phase7.sh`.

**Cost:** 1 live package order (+ optional top-up) deducts Airalo partner balance.
Airalo FAQ allows 1–2 test orders then a support ticket for “testing” refund.

Then confirm staging smoke PASS with Roamkit-Sandbox credentials.

## Evidence close-out

1. Copy redacted artifacts → `api-logs/phase7/`
2. Fill [e2e-evidence.md](./e2e-evidence.md) production section
3. Write [phase-7-complete.md](../releases/1.0.0/evidence/phase-7-complete.md)
4. Update [communications.md](./communications.md) Production confirmation
5. End Production Freeze; unlock [voucher-pr1-kickoff.md](../voucher-pr1-kickoff.md)

Phase 7 is **GREEN** only when production live smoke **and** staging smoke after
sandbox credentials both PASS.

## Failure path (if production live smoke fails)

Same discipline as Gate C / rollback drills:

```text
1. Stop further live orders
2. AIRALO_ENABLED=false on production → recreate api/celery/beat
3. Rollback previous production images if required
   (./scripts/rollback-production.sh + smoke-test-production.sh)
4. Notify Guy (brief status; no further live traffic)
5. Phase 7 remains YELLOW — do not write phase-7-complete GREEN
```

## Abort

- Guy NO-GO / incomplete Go Live checklist → leave production unset for live; do not run live smoke
- Preflight FAIL → **STOP**; do not run `CONFIRM_LIVE_ORDER=1`
- Live smoke FAIL → Failure path above; do not end freeze
- Staging still on live credentials after switch → **STOP ordering on staging** until sandbox pair installed

## Related

- [communications.md](./communications.md)
- [release-decision.md](./release-decision.md)
- [e2e-evidence.md](./e2e-evidence.md)
- Airalo Go Live checklist: https://developers.partners.airalo.com/go-live-checklist-1531786m0
