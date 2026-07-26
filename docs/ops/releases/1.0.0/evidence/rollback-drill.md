# Evidence — Phase 4 Exit Criterion #4 (Rollback drill)

> Rollback drill is additional verification of Exit Criterion #4 only.
> It does **not** reopen Gate C or change the Gate C GO decision (2026-07-25).

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Criterion | Phase 4 Exit #4 — Rollback verified |
| Verdict | **STOP / FAIL** (preflight Go/No-Go) |
| Criterion #4 status | remains **RED** |
| Abort? | **yes** — no rollback or restore commands executed |
| Preflight UTC | 2026-07-26T11:13:37Z |
| Host | `dedicated-hel1` → `/opt/stacks/roamkit-production/` |
| Change Freeze | observed from preflight start through evidence write |

## Rollback Owner

```text
Facilitator: Engineering (solo operator / agent execution)
Observer: —
Approval: Operator GO for Criterion #4 drill only (2026-07-26)
```

## Go / No-Go preflight

| Check | Required | Result | Notes |
|-------|----------|:------:|-------|
| Distinct N i N−1 image | ✅ | ❌ | API current = `.previous-tag` API (`caa3f1d…`). Only WEB differs (`7546edb…` vs `bf62035…`). Not a valid N / N−1 pair for full release rollback. |
| Migration head identical | ✅ | ⏳ | Not evaluated against a distinct API N−1 (blocked by distinct-image ❌). Running DB head tip includes `orders.0003_order_idempotency_key`, `billing.0001_billing_schema`. |
| DB backup exists | ✅ | ✅ | `/var/backups/roamkit_production_20260725T213943Z.sql.gz`, `…T220527Z.sql.gz` |
| `.previous-tag` valid | ✅ | ⚠️ | File present and readable; `API_IMAGE` set; targets **same API** as running N → invalid as N−1 for API. |
| No concurrent deploy | ✅ | ✅ | No `deploy-production` / `rollback-production` process running |
| `traefik.enable=false` | ✅ | ❌ | Live containers: `traefik.enable=true`. Public `https://api.roamkit.net/version` returns 200. Dress-rehearsal isolation assumption false. |

**Preflight decision:** **STOP** — two hard ❌ (distinct images, traefik isolation). Per plan: do not run rollback.

## Running baseline at STOP (release N — unchanged)

| Component | Value |
|-----------|-------|
| API image | `ghcr.io/roamkit-net/roamkit-api:caa3f1d0e3e4c53bacf76719a0f18b1473560207` |
| WEB image | `ghcr.io/roamkit-net/roamkit-web:7546edb2f732a5e2de46cb35b0dab3bc66993f6c` |
| Local `/version` | `{"git_sha":"caa3f1d0e3e4c53bacf76719a0f18b1473560207","build_date":"2026-07-25T21:38:06Z","image_tag":"caa3f1d0e3e4c53bacf76719a0f18b1473560207","environment":"production"}` |
| Public `/version` | same SHA (Traefik serving production) |
| `.previous-tag` | API `caa3f1d…` (same) · WEB `bf62035325279c87153a93d71efc167a9bc5f80d` |

### `docker compose --profile app ps` (at STOP)

```text
roamkit-api-production           …caa3f1d…   Up (healthy)   127.0.0.1:18000->8000/tcp
roamkit-web-production           …7546edb…   Up (healthy)   127.0.0.1:13000->3000/tcp
roamkit-celery-production        …           Up
roamkit-celery-beat-production   …           Up
```

## Expected vs Observed SHA

| Moment | Expected API SHA | Expected WEB SHA | Observed API SHA | Observed WEB SHA | Result |
|--------|------------------|------------------|------------------|------------------|--------|
| After rollback | — | — | — | — | **N/A** (not executed) |
| After restore | — | — | — | — | **N/A** (not executed; N never left) |

## Timestamps + Recovery Time

```text
rollback start UTC:   (not started)
rollback finish UTC:  (not started)
restore start UTC:    (not started)
restore finish UTC:   (not started)

Recovery Time = n/a
Target: <10 min
Actual: n/a
```

## Command log summary

- SSH preflight only (`compose ps`, `images`, `.env` / `.previous-tag` read, `/version`, Traefik label inspect, backup listing, migration tip, public `/version`).
- **Not run:** `rollback-production.sh`, `deploy-production.sh`, `smoke-test-production.sh`.
- **Restore:** not required — stack remained on N.

## Abort reason

1. Plan Go/No-Go requires dress-rehearsal isolation (`traefik.enable=false`). Host is on live Traefik with public `api.roamkit.net` (Gate D artifacts present on host: `gate-d-*.log`, hypercare). Executing image rollback would affect public traffic — outside Criterion #4 dress-rehearsal procedure as written.
2. No distinct API N−1 in `.previous-tag`. A WEB-only rollback would not satisfy “return previous release” for the API.

## Evidence Bundle checklist

| Artefact | Status |
|----------|--------|
| Rollback Owner | ✅ |
| Go/No-Go table | ✅ |
| UTC timestamps / Recovery Time | ✅ (n/a) |
| Expected vs Observed SHA | ✅ (N/A) |
| rollback log | ✅ (none — not run) |
| deploy restore log | ✅ (none — not run) |
| smoke output | ✅ (none — not run) |
| `/version` | ✅ (baseline N) |
| `docker compose ps` | ✅ |
| git / image SHAs N and N−1 | ✅ (N−1 API missing / identical) |
| Abort yes/no + reason | ✅ |
| Lessons Learned | ✅ below |

## Lessons Learned

```text
What worked:
- Preflight caught unsafe conditions before any mutate.
- DB backups and scripts are present; Change Freeze held (no deploy/merge during attempt).
- Public vs localhost /version confirmed Traefik is live.

What should improve:
- Refresh Criterion #4 procedure for post–Gate D reality (live Traefik) OR re-isolate dress-rehearsal before drill.
- Ensure a distinct pullable API+WEB N−1 with identical migration head before Go/No-Go (or document WEB-only drill as insufficient).
- Keep `.previous-tag` in sync with a real prior release pair after every successful deploy.

Actions:
1. Decide: run drill under live Traefik with explicit public-risk approval + maintenance window, OR temporarily disable Traefik on production for a controlled dress-rehearsal window.
2. Establish distinct N−1 API image (same migration head as N) and rewrite `.previous-tag` via a controlled deploy cycle under Change Freeze.
3. Re-run Criterion #4 only after Go/No-Go is all ✅. Leave #4 RED until then.
```

## Related

- [disaster-day.md](../../../disaster-day.md)
- [gate-c.md](./gate-c.md) (unchanged GO)
- Plan: `.cursor/plans/roamkit.plan.md` — Exit Criterion #4
