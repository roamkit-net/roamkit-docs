# Disaster Day

Annual failure simulation **after** stable production. Not a Gate D hard blocker
(see Launch Gates C8 remap). Still required before declaring long-term go-live polish complete.

## Scenarios

- [ ] Postgres unavailable / restart
- [ ] Polygon RPC provider down / slow
- [ ] Disk full on app host
- [ ] Deploy **rollback** drill
- [ ] Backup **restore** to scratch DB (prove restore, not only backup job)

## Record

| Field | Value |
|-------|-------|
| Date (UTC) | |
| Facilitator | |
| Scenarios run | |
| Recovery time (per scenario) | |
| Gaps / actions | |

Store notes under `releases/` or a dated file in this folder (e.g. `disaster-day-YYYY.md`).

## Related

- [Migration Ready](./migration-ready.md) (backup/restore discipline)
- [Incident runbook](./incident-runbook.md)
- [Launch Gates](./launch-gates.md)
