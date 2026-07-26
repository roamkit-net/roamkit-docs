# Evidence — Phase 4 Exit Criterion #3 (Billing E2E) — PASS

| Field | Value |
|-------|-------|
| Criterion | Phase 4 Exit #3 — Billing E2E (public) |
| Verdict | **PASS** → Criterion #3 **YELLOW → GREEN** |
| Closed at (UTC) | 2026-07-26T11:46:20Z |
| Inventory | [billing-e2e-criterion-3-inventory.md](./billing-e2e-criterion-3-inventory.md) |
| Prior Gate C | [billing-e2e.md](./billing-e2e.md) (localhost dress-rehearsal) |
| Gate C | **GO** (not reopened) |

## Facilitator

```text
Facilitator: Engineering (solo operator / agent execution)
Approval: Operator GO for Criterion #3 (2026-07-26)
```

## DoD

| Requirement | Result | Evidence |
|-------------|:------:|----------|
| Public `https://api.roamkit.net` | ✅ | `/version` = `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| Billing env gate | ✅ | `billing=True wc=False chain=137` |
| `production-dod-billing.sh` vs public API | ✅ | `PRODUCTION BILLING DoD PASSED` |
| Deposit stand-in → ledger → balance → order | ✅ | credit 5.000000 → order id=6 fulfilled → balance 4.000000 |
| verify-cex negative path | ✅ | HTTP 400 |
| `GET /api/v1/billing/config/` on production host | ✅ | 200; `token_symbol=USDT`, `display_decimals=2` |
| Evidence + SHA/date | ✅ | this file |

### Run summary

```text
Gate C production billing DoD against https://api.roamkit.net
env OK billing=True wc=False chain=137 wallet=0x7F803870…
deposit-info OK
balance OK 0
Using package discover-in-3days-300mb
order 402 OK (insufficient credits)
verify-cex negative path OK (HTTP 400)
Admin-adjust credit 5.000000 (deposit stand-in)...
ledger==balance OK 5.000000
API balance OK 5.000000
order OK id=6 status=fulfilled
balance/ledger after spend: 4.000000
PRODUCTION BILLING DoD PASSED
```

| Field | Value |
|-------|-------|
| Start UTC | 2026-07-26T11:46:05Z |
| Finish UTC | 2026-07-26T11:46:20Z |
| API SHA | `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| Script | `/opt/stacks/roamkit-production/scripts/production-dod-billing.sh` |
| API_URL | `https://api.roamkit.net` |
| WEB_URL | `https://roamkit.net` |
| Deposit path | `CreditService.admin_adjust` stand-in (no `VERIFY_TX_HASH`) |

## Catalog UI note (not a DoD-script failure)

- Public `/global-esim` returns 200; SSR still shows `catalog-price-skeleton` until client hydrate.
- Running web image (`7064a9b…`) **bakes** `api.staging.roamkit.net` (develop Dockerfile default). Runtime `NEXT_PUBLIC_API_URL=https://api.roamkit.net` does not rewrite client chunks.
- Production bake (`api.roamkit.net`) is produced on `main` builds per `roamkit-web` docker workflow — **backlog for Criterion #1 / production web image**, not Criterion #3 API Billing DoD.

## Lessons Learned

```text
What worked:
- Same production-dod-billing.sh against public HTTPS API without code changes.
- Packages synced (1983); money path green on live Traefik.

What should improve:
- Deploy a main-built web image with NEXT_PUBLIC_API_URL=https://api.roamkit.net before declaring catalog UI prices Observed.
- Optional: real VERIFY_TX_HASH path once a known Polygon USDT tx is available (not required for this GREEN).

Actions (backlog):
1. Criterion #1 — production web bake + catalog price visibility after hydrate.
2. Optional on-chain verify drill.
```

## Out of scope

Gate D cutover · Traefik SNI · Airalo · new billing features · app notifications.
