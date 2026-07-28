# Evidence — Gate C billing E2E

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Gate | C |
| Milestone | Billing E2E (`production-dod-billing.sh`) |
| Verdict | GO (dress-rehearsal localhost URLs) |
| Closed at (UTC) | 2026-07-25T21:39:22Z |
| GO by (role / name) | Engineering (solo operator) |
| API SHA | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` |
| Script | `/opt/stacks/roamkit-production/scripts/production-dod-billing.sh` |
| API_URL | `http://127.0.0.1:18000` |
| WEB_URL | `http://127.0.0.1:13000` |

## Result

```text
Gate C production billing DoD against http://127.0.0.1:18000
env OK billing=True wc=False chain=137
deposit-info OK
balance OK 0
order 402 OK (insufficient credits)
verify-cex negative path OK (HTTP 400)
Admin-adjust credit 5.000000 (deposit stand-in)
ledger==balance OK 5.000000
API balance OK 5.000000
order OK id=2 status=fulfilled
balance/ledger after spend: 4.000000
PRODUCTION BILLING DoD PASSED
```

## Notes

- Deposit stand-in via `CreditService.admin_adjust` (script default path).
- Real on-chain `VERIFY_TX_HASH` not used in this run.
- Catalog prerequisite: `sync_packages` → 1983 packages before DoD.
