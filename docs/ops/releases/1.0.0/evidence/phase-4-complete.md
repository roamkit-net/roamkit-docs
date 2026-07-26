# Evidence — Phase 4 Complete

| Field | Value |
|-------|-------|
| Program phase | Phase 4 — Production Infrastructure Readiness (ADR 013) |
| Closed at (UTC) | 2026-07-26T22:38:24Z (Hypercare end) |
| Re-verified at (UTC) | 2026-07-26T23:00:00Z (close-out agent) |
| Verdict | **GREEN** — all Exit Criteria closed with merged evidence |
| Next focus | **Phase 5 — Airalo Go-Live Readiness** |

## Exit Criteria scorecard (final)

| # | Criterion | Status | Evidence |
|---|-----------|:------:|----------|
| 4 | Rollback verified | 🟢 GREEN | [rollback-drill.md](./rollback-drill.md) |
| 2 | Observability complete | 🟢 GREEN | [observability-criterion-2.md](./observability-criterion-2.md) |
| 3 | Billing E2E (public) | 🟢 GREEN | [billing-e2e-criterion-3.md](./billing-e2e-criterion-3.md) |
| 1 | Production stack / Gate D | 🟢 GREEN | [gate-d-criterion-1.md](./gate-d-criterion-1.md) · [gate-d.md](./gate-d.md) |

## Launch Gates

| Gate | Verdict |
|------|---------|
| A–C | Closed (prior) |
| D | **Unconditional GO** at Hypercare exit — [release-decision-log.md](../../../release-decision-log.md) |

## Close-out re-verify (2026-07-26T23:00Z)

| Check | Result |
|-------|--------|
| Hypercare clock | Complete (`elapsed≈24.37h`; host `HYPERCARE_COMPLETE`) |
| Transient Hypercare ALERTs | 2 single-sample 404 blips (07:53Z, 19:23Z); recovered next sample; **not** unresolved P0/P1 |
| Public smoke | live/ready/version/billing-config/web/global-esim → **200** |
| Catalog Playwright | happy path **PASS** on `https://roamkit.net` |
| Web bake | `api.roamkit.net` present; `api.staging` absent in running image |
| Kuma 13–17 | active |
| Rollback for Hypercare incidents | **Not required** |

## Running production (at re-verify)

| Component | Image / SHA |
|-----------|-------------|
| API | `ghcr.io/roamkit-net/roamkit-api:main` (OCI rev `631e79b…`) |
| Web | `ghcr.io/roamkit-net/roamkit-web:e91664d67fcbf67f613af0b67eedda1072cfc3ef` (production bake) |

See [manifest.md](../manifest.md) and [gate-d.md](./gate-d.md) Success Snapshot.

## Phase 5 handoff

Phase 4 platform exit is **GREEN**. Formal focus moves to [Airalo Go-Live Readiness](../../../airalo-go-live-readiness/README.md) (Phase 5). Do not reopen Phase 4 Exit Criteria unless a production P0 forces Gate D incident process.
