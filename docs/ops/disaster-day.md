# Disaster Day

Annual failure simulation **after** stable production. Not a Gate D hard blocker
(see Launch Gates C8 remap). Still required before declaring long-term go-live polish complete.

## Scenarios

- [ ] Postgres unavailable / restart
- [ ] Polygon RPC provider down / slow
- [ ] Disk full on app host
- [x] Deploy **rollback** drill — **PASS** 2026-07-26T11:32:20Z–11:33:11Z ([rollback-drill.md](./releases/1.0.0/evidence/rollback-drill.md))
- [ ] Backup **restore** to scratch DB (prove restore, not only backup job)

## Record

| Field | Value |
|-------|-------|
| Date (UTC) | 2026-07-26T11:32:20Z (rollback start) |
| Facilitator | Engineering (solo operator / agent execution) |
| Scenarios run | Deploy rollback drill (GO #2) — **PASS** |
| Recovery time (per scenario) | **13s** (rollback start→finish); target <10 min |
| Gaps / actions | Optional: sync `ROAMKIT_*` in deploy/rollback so `/version` matches image without manual `.env` align. Full Disaster Day scenarios still open. |

### Prior attempts (audit)

1. STOP Go/No-Go — [rollback-drill.md](./releases/1.0.0/evidence/rollback-drill.md) history / [rollback-drill-retry.md](./releases/1.0.0/evidence/rollback-drill-retry.md)
2. Prereqs + qualification — [rollback-retry-prerequisites.md](./releases/1.0.0/evidence/rollback-retry-prerequisites.md) / [rollback-candidate-qualification.md](./rollback-candidate-qualification.md)
3. GO #1 establish N — [rollback-n-established.md](./releases/1.0.0/evidence/rollback-n-established.md)
4. **GO #2 PASS** — [rollback-drill.md](./releases/1.0.0/evidence/rollback-drill.md) / [rollback-drill-pass.md](./releases/1.0.0/evidence/rollback-drill-pass.md)

Store notes under `releases/` or a dated file in this folder (e.g. `disaster-day-YYYY.md`).

## Related

- [Migration Ready](./migration-ready.md) (backup/restore discipline)
- [Incident runbook](./incident-runbook.md)
- [Launch Gates](./launch-gates.md)
- [Rollback candidate qualification](./rollback-candidate-qualification.md)
- [Rollback drill PASS](./releases/1.0.0/evidence/rollback-drill.md)
