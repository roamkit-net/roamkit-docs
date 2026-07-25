# Gate D cutover runbook

**Entry:** Gate C = GO · Production Freeze active · Release Manifest draft ready.  
**Exit:** Hypercare complete · Manifest filled · Freeze closed.

Follow [Launch Gates](./launch-gates.md) time-boxes unless Manifest overrides.

## Time-box (defaults)

| Window | Default |
|--------|---------|
| Deployment window | 09:00–11:00 UTC (weekday) |
| Rollback window | 30 minutes after cutover |
| Hypercare | 24 hours |

## Pre-flight (same day)

- [ ] Production Freeze announced / enforced on `develop`
- [ ] [Migration Ready](./migration-ready.md) signed
- [ ] Backup confirmed
- [ ] Feature flag matrix prepared (ADR 013): billing ON, WC OFF, subscriptions OFF, vouchers OFF
- [ ] Rollback owner + command known
- [ ] [Release Manifest](./release-manifest.md) draft open for edit

## Cutover steps

1. Deploy production images (`main` / agreed SHA) within deployment window.
2. Migrate · health · `/version` (non-empty `git_sha`).
3. Point DNS / Traefik: `roamkit.net`, `www`, `api.roamkit.net` → production stack; staging remains `staging.*` only.
4. Apply prod `.env` flags; record in Manifest.
5. Smoke + one manual deposit/buy if credentials allow.
6. Start **rollback window** clock (30 min).
7. Enter **Hypercare** (24h): watch Sentry, uptime, [billing dashboard](./billing-dashboard.md) metrics, reconcile alerts.

## Rollback window

If P0 within 30 minutes → [incident-runbook.md](./incident-runbook.md) rollback tree; defer Gate D exit.

## After Hypercare

1. Close freeze (if still held).
2. Commit filled Manifest under `releases/X.Y.Z.md`.
3. Run [production-retrospective.md](./production-retrospective.md).

## Related

- [Launch Gates](./launch-gates.md)
- [Go-live checklist](./production-go-live-checklist.md)
- ADR 013
