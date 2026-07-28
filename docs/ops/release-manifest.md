# Release Manifest

One filled manifest per **production** release. Template below; store completed
copies under [`releases/<version>/manifest.md`](./releases/). Gate close audit packs:
[`evidence-of-gate.md`](./evidence-of-gate.md).

## Rules

- Create the draft during **Gate C exit**.
- Fill and commit (or attach to sign-off) at **Gate D** cutover.
- `git_sha` and image refs must match what `/version` and GHCR report.
- Record feature flags exactly as running in production `.env`.

## Template

```markdown
# Release X.Y.Z

| Field | Value |
|-------|-------|
| Release | X.Y.Z |
| Git SHA | |
| Docker image (API) | ghcr.io/roamkit-net/roamkit-api: |
| Docker image (web) | ghcr.io/roamkit-net/roamkit-web: |
| Migration version | (django showmigrations / latest billing) |
| ADR baseline | 010, 012, 013 Accepted; 007 Superseded |
| Feature flags | BILLING_ENABLED= · WALLETCONNECT_ENABLED= · SUBSCRIPTIONS_ENABLED= · VOUCHERS_ENABLED= |
| Rollback version | previous image tag / `.previous-tag` |
| Deploy date (UTC) | |
| Operator | |
| Deployment window | 09:00–11:00 UTC (or override) |
| Rollback window | 30 minutes |
| Hypercare until (UTC) | |
```

## Related

- [Launch Gates](./launch-gates.md)
- [Evidence of Gate](./evidence-of-gate.md)
- [Gate D cutover](./gate-d-cutover.md)
- Infra contract: `ROAMKIT_*` env → `GET /version`
