# Disaster Day

Annual failure simulation **after** stable production. Not a Gate D hard blocker
(see Launch Gates C8 remap). Still required before declaring long-term go-live polish complete.

## Scenarios

- [ ] Postgres unavailable / restart
- [ ] Polygon RPC provider down / slow
- [ ] Disk full on app host
- [ ] Deploy **rollback** drill — attempts 2026-07-26: (1) STOP Go/No-Go; (2) retry STOP before mutate — Criterion #4 remains RED
- [ ] Backup **restore** to scratch DB (prove restore, not only backup job)

## Record

| Field | Value |
|-------|-------|
| Date (UTC) | 2026-07-26 (attempt 1 ~preflight; attempt 2 11:24:04Z) |
| Facilitator | Engineering (solo operator / agent execution) |
| Scenarios run | Deploy rollback drill — **aborted twice before mutate** |
| Recovery time (per scenario) | n/a |
| Gaps / actions | (1) no distinct API N−1 + live Traefik vs dress-rehearsal — [rollback-drill.md](./releases/1.0.0/evidence/rollback-drill.md). (2) N−1 `86ab449` lacks `GET /version` — [rollback-drill-retry.md](./releases/1.0.0/evidence/rollback-drill-retry.md). Next: N−1 must include `/version`; use newer same-mig SHA as drill N. |

Store notes under `releases/` or a dated file in this folder (e.g. `disaster-day-YYYY.md`).

## Related

- [Migration Ready](./migration-ready.md) (backup/restore discipline)
- [Incident runbook](./incident-runbook.md)
- [Launch Gates](./launch-gates.md)
- [Rollback drill STOP](./releases/1.0.0/evidence/rollback-drill.md)
- [Rollback drill retry STOP](./releases/1.0.0/evidence/rollback-drill-retry.md)
- [Retry prerequisites](./releases/1.0.0/evidence/rollback-retry-prerequisites.md)
