# Release 1.0.0

Filled at Gate D cutover; updated at Hypercare exit. See [release-manifest.md](../../release-manifest.md).

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Git SHA (API OCI revision) | `631e79b9b9009f952984f5791bf7e5d919631b60` |
| Git SHA (web running) | `e91664d67fcbf67f613af0b67eedda1072cfc3ef` |
| Docker image (API) | `ghcr.io/roamkit-net/roamkit-api:main` (`sha256:0be6dfe1a60240c761d3488e6aba4cac920ee5119b85f2213ebf95b9f42bd3ac`, OCI rev `631e79b…`) |
| Docker image (web) | `ghcr.io/roamkit-net/roamkit-web:e91664d67fcbf67f613af0b67eedda1072cfc3ef` |
| `/version` note | Env `ROAMKIT_GIT_SHA` currently mirrors web SHA — fix tracked in retrospective |
| Migration version | billing + catalog/orders/esims heads applied (`esims.0003_esim_lifecycle_wave1` present) |
| ADR baseline | 010, 011, 012, 013 Accepted; 007 Superseded |
| Feature flags | BILLING_ENABLED=true · WALLETCONNECT_ENABLED=false · SUBSCRIPTIONS_ENABLED=false · VOUCHERS_ENABLED=false |
| Rollback version (`.previous-tag`) | API `631e79b9b9009f952984f5791bf7e5d919631b60` · WEB `51fae91d0e7c51211301f6b3a1db17cf9b840429` _(file at close-out; earlier cutover N was `caa3f1d…` / `7546edb…`)_ |
| Deploy date (UTC) | Cutover `2026-07-25T22:05:27Z` |
| Operator | Engineering / Operations (solo operator) |
| Deployment window | Cutover outside default 09:00–11:00 UTC weekday window (recorded) |
| Rollback window | 30 minutes — **COMPLETE** `2026-07-25T22:37:56Z` |
| Hypercare until (UTC) | `2026-07-26T22:38:24Z` — **COMPLETE** |

Status: **Gate D = GO** — Hypercare closed; Production Freeze lifted for normal merge discipline.

## Evidence packs

| Gate / milestone | Status | File |
|------------------|--------|------|
| Gate C — API slice | recorded | [evidence/gate-c-api.md](./evidence/gate-c-api.md) |
| Gate C — full GO | **GO** | [evidence/gate-c.md](./evidence/gate-c.md) |
| Gate D — inventory | recorded | [evidence/gate-d-criterion-1-inventory.md](./evidence/gate-d-criterion-1-inventory.md) |
| Gate D — execution | EXECUTE PASS; hypercare closed | [evidence/gate-d-criterion-1.md](./evidence/gate-d-criterion-1.md) |
| Gate D — close-out | **GO** | [evidence/gate-d.md](./evidence/gate-d.md) |
| Phase 4 Complete | **GREEN** | [evidence/phase-4-complete.md](./evidence/phase-4-complete.md) |
| Retrospective | filled | [1.0.0-retro.md](../1.0.0-retro.md) |
