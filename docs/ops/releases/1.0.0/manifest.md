# Release 1.0.0

Filled at Gate D execution (Phase 4 Exit Criterion #1). See [release-manifest.md](../../release-manifest.md).

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Git SHA (API running) | `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| Git SHA (web running) | `7064a9b953ad167e7a23dca6f454ca66b09482de` |
| Docker image (API) | `ghcr.io/roamkit-net/roamkit-api:9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| Docker image (web) | `ghcr.io/roamkit-net/roamkit-web:7064a9b953ad167e7a23dca6f454ca66b09482de` (`sha256:732856f00bb3bafe00f03e0bfead5892081759d5d226871665c2ae3fc3eeaa09`, production bake) |
| Web `main` tip (baked, not yet running) | `8afd7268f55ecc63b1684010582521da036be4fd` |
| Migration version | `billing.0001_billing_schema` applied (+ catalog/orders/esims heads present on prod) |
| ADR baseline | 010, 011, 012, 013 Accepted; 007 Superseded |
| Feature flags | BILLING_ENABLED=true · WALLETCONNECT_ENABLED=false · SUBSCRIPTIONS_ENABLED=false · VOUCHERS_ENABLED=false |
| Rollback version (`.previous-tag`) | API `caa3f1d0e3e4c53bacf76719a0f18b1473560207` · WEB `7546edb2f732a5e2de46cb35b0dab3bc66993f6c` |
| Deploy date (UTC) | Cutover `2026-07-25T22:05:27Z`; production web bake redeploy `2026-07-26T12:00:00Z` (approx) |
| Operator | Engineering (solo operator) |
| Deployment window | Cutover outside default 09:00–11:00 UTC weekday window (recorded) |
| Rollback window | 30 minutes — **COMPLETE** `2026-07-25T22:37:56Z` |
| Hypercare until (UTC) | `2026-07-26T22:38:24Z` |

Status: **Gate D GO WITH CONDITIONS** — technical cutover + catalog bake PASS; Hypercare clock open until row above.

## Evidence packs

| Gate / milestone | Status | File |
|------------------|--------|------|
| Gate C — API slice | recorded | [evidence/gate-c-api.md](./evidence/gate-c-api.md) |
| Gate C — full GO | **GO** | [evidence/gate-c.md](./evidence/gate-c.md) |
| Gate D — inventory | recorded | [evidence/gate-d-criterion-1-inventory.md](./evidence/gate-d-criterion-1-inventory.md) |
| Gate D — execution | EXECUTE PASS; hypercare open | [evidence/gate-d-criterion-1.md](./evidence/gate-d-criterion-1.md) |
| Gate D — close-out | pending hypercare end | — |
