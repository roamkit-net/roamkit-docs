# Migration Ready

Gate C / pre-deploy check before applying migrations on production.

## Checklist

- [ ] Migrations are **reversible** where possible (`ReverseMigration` / documented no-op reverse)
- [ ] Estimated **duration** recorded (staging dry-run timing OK)
- [ ] Estimated **lock time** / table locks understood (esp. billing / Account)
- [ ] **Backup** taken (or confirmed recent) before migrate
- [ ] **Rollback path** documented (migrate reverse and/or image rollback via `.previous-tag`)
- [ ] No money-path data rewrite outside migrations

## Sign-off

| Field | Value |
|-------|-------|
| Release | |
| Migration tip | |
| Staging dry-run duration | |
| Backup ID / path / time | |
| Operator | |
| Date (UTC) | |

**NO-GO** if backup missing or lock risk unknown for billing tables.

## Related

- [Launch Gates](./launch-gates.md) — Gate C exit
- [Release Manifest](./release-manifest.md)
- AGENTS.md database rules
