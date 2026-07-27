# Phase 7 — Airalo Production Switch

Partner-side cutover from Sandbox → Production for Fine Star / RoamKit.
Same API credentials and base URL (`https://partners-api.airalo.com`); mode is
controlled by Airalo on the partner account ([Sandbox mode docs](https://developers.partners.airalo.com/sandbox-mode-2114305m0)).

## Preconditions

- [release-decision.md](./release-decision.md) = **GO**
- Phase 6 Pilot 20 (+ Pilot 100) GREEN
- Production Request DoD checklist complete
- Staging and production currently share the same Airalo `client_id` → **must**
  obtain a **separate sandbox credential set for staging** before or at switch
  (otherwise staging fulfillment becomes live)

## Sequence

```text
1. Send Production request to Guy (communications.md)
2. Complete Partner Platform Go Live checklist
3. Guy / Airalo approves → switches account to Production mode
4. (Preferred) Receive separate sandbox credentials → install on staging only
5. Production .env: AIRALO_SANDBOX=false → recreate api/celery/beat
6. CONFIRM_LIVE_ORDER=1 ./scripts/production-dod-airalo-phase7.sh
7. Record evidence → end Production Freeze
```

## Production request

Draft / sent status: [communications.md](./communications.md).

To: `guy.dor@airalo.com` · From ops mailbox `info@roamkit.net` · CC operator.

Ask for:

1. Switch Fine Star (Application Id **17558**) Partner API account to **Production**
2. Separate **sandbox** credentials for RoamKit staging (`api.staging.roamkit.net`)
3. Confirmation when Production mode is active

## Staging pause (before Partner Production flip)

Staging currently shares the live `client_id`. **Before** Guy flips Production mode:

```bash
cd /opt/stacks/roamkit-net
./scripts/staging-pause-airalo.sh
```

After dedicated sandbox credentials arrive:

```bash
./scripts/staging-pause-airalo.sh restore   # restores backup keys
# then set the new sandbox AIRALO_CLIENT_ID / AIRALO_CLIENT_SECRET in .env
docker compose --profile app up -d --force-recreate api celery celery-beat
```

## Flip production env

On `/opt/stacks/roamkit-production/`:

```bash
# After Guy confirms Production mode is active:
sed -i 's/^AIRALO_SANDBOX=.*/AIRALO_SANDBOX=false/' .env
grep '^AIRALO_' .env | sed -E 's/(CLIENT_ID|CLIENT_SECRET)=.*/\1=***REDACTED***/'
docker compose --profile app up -d --force-recreate api celery celery-beat
docker compose --profile app exec -T api python -c \
  'from django.conf import settings; assert settings.AIRALO_SANDBOX is False; print("OK")'
```

Staging must keep `AIRALO_SANDBOX=true` **and** use the dedicated sandbox
`client_id` / `client_secret` (not the live pair).

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

## Evidence close-out

1. Copy redacted artifacts → `api-logs/phase7/`
2. Fill [e2e-evidence.md](./e2e-evidence.md) production section
3. Write [phase-7-complete.md](../releases/1.0.0/evidence/phase-7-complete.md)
4. Update [communications.md](./communications.md) Production confirmation
5. End Production Freeze; unlock [voucher-pr1-kickoff.md](../voucher-pr1-kickoff.md)

## Abort

- Guy NO-GO / incomplete Go Live checklist → leave `AIRALO_SANDBOX=true`; do not run live smoke
- Live smoke FAIL → stop, file P0/P1, do not end freeze
- Staging still on live credentials after switch → **STOP ordering on staging** until sandbox pair installed

## Related

- [communications.md](./communications.md)
- [release-decision.md](./release-decision.md)
- [e2e-evidence.md](./e2e-evidence.md)
- Airalo Go Live checklist: https://developers.partners.airalo.com/go-live-checklist-1531786m0
