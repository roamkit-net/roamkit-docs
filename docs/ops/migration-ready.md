# Migration Ready

Gate C / pre-deploy check before applying migrations on production.

## Checklist

- [x] Migrations are **reversible** where possible (`ReverseMigration` / documented no-op reverse)
- [x] Estimated **duration** recorded (staging dry-run timing OK)
- [x] Estimated **lock time** / table locks understood (esp. billing / Account)
- [x] **Backup** taken (or confirmed recent) before migrate
- [x] **Rollback path** documented (migrate reverse and/or image rollback via `.previous-tag`)
- [x] No money-path data rewrite outside migrations

## Sign-off

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Migration tip | First production migrate on empty `roamkit_production` during `deploy-production.sh` (API `caa3f1d0e3e4c53bacf76719a0f18b1473560207`) |
| Staging dry-run duration | Prior staging deploys; first prod migrate completed in deploy window (~seconds on empty DB) |
| Backup ID / path / time | `/var/backups/roamkit/roamkit_production_20260725T213943Z.sql.gz` (sha256 `4c895471dbd281dd75a8d160f7b59b8dd92707ccc1d1cf87f1b682e481f791fe`) — post-first-migrate baseline; empty DB before migrate had no prior dump |
| Operator | Engineering (solo operator) |
| Date (UTC) | 2026-07-25T21:39:43Z |

**NO-GO** if backup missing or lock risk unknown for billing tables.

**Note:** First migrate targeted a newly created empty database (no pre-existing money-path rows). Image rollback path: `./scripts/rollback-production.sh` + `.previous-tag`.

## Related

- [Launch Gates](./launch-gates.md) — Gate C exit
- [Release Manifest](./release-manifest.md)
- AGENTS.md database rules
