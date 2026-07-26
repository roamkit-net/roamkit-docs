# Evidence — Gate D (Customer Traffic) — full GO

**Index + Success Snapshot.** Cutover execution detail: [gate-d-criterion-1.md](./gate-d-criterion-1.md).

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Gate | D |
| Milestone | Customer Traffic / Hypercare exit |
| Verdict | **GO** |
| Closed at (UTC) | 2026-07-26T22:38:24Z |
| GO by (role / name) | Operations / Release (solo operator) |
| Cutover start (UTC) | 2026-07-25T22:05:27Z |
| Rollback window | **COMPLETE** 2026-07-25T22:37:56Z (no abort) |
| Hypercare | 2026-07-25T22:38:24Z → 2026-07-26T22:38:24Z · 96 samples · 2 transient ALERTs · no unresolved P0/P1 |
| Decision Log Entry | [release-decision-log.md](../../../release-decision-log.md) — Gate D GO |
| Related | [gate-d-cutover.md](../../../gate-d-cutover.md) · [gate-d-criterion-1.md](./gate-d-criterion-1.md) · [manifest.md](../manifest.md) |

## Abort Criteria (applied)

| Događaj | Outcome |
|---------|---------|
| P0 tijekom cutovera | Did not occur — no immediate rollback |
| P1 tijekom rollback windowa | Did not occur — window COMPLETE |
| DNS / routing problem | Resolved at cutover (`api.roamkit.net` A created; Traefik Hosts flipped) |

## GO checklist (evidence index)

| Check | Status | Evidence |
|-------|:------:|----------|
| Criterion #1 execution | ✅ | [gate-d-criterion-1.md](./gate-d-criterion-1.md) |
| Rollback window 30m | ✅ | Host `gate-d-rollback-window.log` — COMPLETE |
| Hypercare 24h | ✅ | Host `gate-d-hypercare.log` — `HYPERCARE_COMPLETE` |
| Success Snapshot | ✅ | below |
| Public smoke (post-hypercare) | ✅ | live/ready/version/web = 200 @ close-out |
| Verdict | **GO** | Release Decision Log |

## Success Snapshot

Taken at Hypercare end (`2026-07-26T22:38:24Z`) from host `gate-d-success-snapshot.txt`.

| Stavka | Vrijednost |
|--------|------------|
| Release SHA (API OCI revision) | `631e79b9b9009f952984f5791bf7e5d919631b60` (`ghcr.io/roamkit-net/roamkit-api:main`) |
| Release SHA (web) | `e91664d67fcbf67f613af0b67eedda1072cfc3ef` |
| `/version` git_sha (env) | `e91664d…` — **drift**: prod `.env` `ROAMKIT_GIT_SHA` / `ROAMKIT_IMAGE_TAG` mirror web SHA; OCI label is API `631e79b…` (fix in retrospective) |
| Active users | 4 |
| Billing success | `failed_verify_count=0` · `total_spent_credits=3.000000` (orders) · `deposit_count=0` on-chain |
| Open Sentry issues | Not queried via API at close-out; no P0/P1 declared in Hypercare; reconcile `drift_hits=0` throughout samples |
| Uptime | Kuma monitors 13–16 heartbeat status **UP** @ 22:34Z; Hypercare public probes 96 samples (2 transient non-200, recovered) |

### Hypercare ALERT notes (not blockers)

1. `2026-07-26T07:53:38Z` — ready/version/web=404 (live=200); recovered next sample.
2. `2026-07-26T19:23:58Z` — live/ready/version=404 (web=200); recovered next sample.

## Related

- [gate-d-cutover.md](../../../gate-d-cutover.md)
- [production-retrospective](../../production-retrospective.md) · dated copy [1.0.0-retro.md](../../1.0.0-retro.md)
