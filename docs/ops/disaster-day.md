# Disaster Day

Annual failure simulation **after** stable production. Not a Gate D hard blocker
(see Launch Gates C8 remap). Still required before declaring long-term go-live polish complete.

## Scenarios

- [ ] Postgres unavailable / restart
- [ ] Polygon RPC provider down / slow
- [ ] Disk full on app host
- [ ] Deploy **rollback** drill — **attempted 2026-07-26; STOP at Go/No-Go** (see evidence; Criterion #4 remains RED)
- [ ] Backup **restore** to scratch DB (prove restore, not only backup job)

## Record

| Field | Value |
|-------|-------|
| Date (UTC) | 2026-07-26T11:13:37Z (preflight STOP) |
| Facilitator | Engineering (solo operator / agent execution) |
| Scenarios run | Deploy rollback drill — **aborted at Go/No-Go** (no rollback executed) |
| Recovery time (per scenario) | n/a (drill not started) |
| Gaps / actions | Traefik live (`traefik.enable=true` + public `api.roamkit.net`); API N = `.previous-tag` API (no distinct API N−1). Re-run after Go/No-Go all ✅. Evidence: [rollback-drill.md](./releases/1.0.0/evidence/rollback-drill.md) |

Store notes under `releases/` or a dated file in this folder (e.g. `disaster-day-YYYY.md`).

## Related

- [Migration Ready](./migration-ready.md) (backup/restore discipline)
- [Incident runbook](./incident-runbook.md)
- [Launch Gates](./launch-gates.md)
- [Rollback drill evidence (STOP)](./releases/1.0.0/evidence/rollback-drill.md)
