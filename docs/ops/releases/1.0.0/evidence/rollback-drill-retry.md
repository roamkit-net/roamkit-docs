# Evidence — Phase 4 Exit Criterion #4 (Rollback drill retry)

> Does **not** reopen Gate C. Criterion #4 remains **RED** until a PASS drill.

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Attempt | Retry after [prerequisites](./rollback-retry-prerequisites.md) |
| Verdict | **FAIL / STOP** (before rollback mutate) |
| Criterion #4 | remains **RED** |
| Abort? | **yes** — `rollback-production.sh` **not** executed |
| Preflight UTC | 2026-07-26T11:24:04Z |
| Traefik mode | **live** (safeguards acknowledged) |
| Prior STOP | [rollback-drill.md](./rollback-drill.md) |

## Rollback Owner

```text
Facilitator: Engineering (solo operator / agent execution)
Observer: —
Approval: Operator GO for Criterion #4 retry (2026-07-26)
```

## Go / No-Go (listed checks)

| Check | Result | Notes |
|-------|:------:|-------|
| Distinct N / N−1 | ✅ | API+WEB differ |
| Migration head identical | ✅ | `9cd77fa6…` |
| DB backup exists | ✅ | `/var/backups/roamkit/roamkit_production_20260725T*.sql.gz` |
| `.previous-tag` → N−1 | ✅ | `86ab449…` / `bf62035…` |
| No concurrent deploy | ✅ | |
| Traefik live mode | ✅ | `traefik.enable=true`; public `/version` = N |

**Listed Go/No-Go:** PASS.  
**Extended preflight (smoke DoD feasibility):** **FAIL** — see Abort.

## Documented targets at attempt

| Role | API SHA | WEB SHA |
|------|---------|---------|
| N (running) | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` | `7546edb2f732a5e2de46cb35b0dab3bc66993f6c` |
| N−1 (planned rollback) | `86ab449e6975b81a448cd0149fec6bd9a85b0662` | `bf62035325279c87153a93d71efc167a9bc5f80d` |

Running `/version` (localhost + public) confirmed N. Stack unchanged from prerequisites.

## Abort reason (before first rollback command)

`smoke-test-production.sh` **hard-requires** `GET /version` with non-empty `git_sha` (Gate C must-have).

N−1 API image `86ab449…` **does not expose** `/version` (urlpatterns: health + api only; `/version` added in `caa3f1d` Gate C PR #21).

Therefore a rollback to documented N−1 would **guaranteed-fail** production smoke on live Traefik, triggering Abort mid-window and public impact for a known-unwinnable DoD.

Per Abort Conditions / Change Freeze: **STOP** without mutate; leave stack on N; Criterion #4 **RED**.

## Expected vs Observed SHA

| Moment | Expected API | Expected WEB | Observed | Result |
|--------|--------------|--------------|----------|--------|
| After rollback | — | — | — | **N/A** (not executed) |
| After restore | — | — | — | **N/A** (N never left) |

## Timestamps + Recovery Time

```text
rollback start UTC:   (not started)
rollback finish UTC:  (not started)
restore start UTC:    (not started)
restore finish UTC:   (not started)
Recovery Time: n/a
Target: <10 min
```

## Command log

- Go/No-Go + image inspect of N−1 urls (`86ab449` has no `path("version")`).
- **Not run:** `rollback-production.sh`, `deploy-production.sh`, `smoke-test-production.sh`.

## Evidence Bundle

| Artefact | Status |
|----------|--------|
| Owner / Go/No-Go / Abort | ✅ |
| Timestamps / Recovery | ✅ n/a |
| SHA table | ✅ N/A |
| Logs / smoke / ps | ✅ baseline only; no mutate |
| Lessons Learned | ✅ |

## Lessons Learned

```text
What worked:
- Listed Go/No-Go passed; extended check caught unwinnable smoke before live rollback.
- Prerequisites correctly fixed distinct N−1 + migration parity + live Traefik mode.

What should improve:
- Go/No-Go must include: "N−1 API serves GET /version (smoke DoD)" when using smoke-test-production.sh.
- Prefer N−1 = last known-good **with** /version; establish drill N as a *newer* same-migration-head image (or accept a drill smoke profile that verifies image tags when rolling to pre-/version builds — not chosen here).

Actions:
1. Redefine targets: N−1 = caa3f1d… (has /version); N = newer API SHA with identical migration hash (e.g. 9989fe… / fe598ab…) + matching WEB pair.
2. Under Change Freeze: deploy that N (writes .previous-tag → current), then re-run the same drill procedure.
3. Add "/version on N−1" to Go/No-Go before next GO.
```

## Related

- [rollback-retry-prerequisites.md](./rollback-retry-prerequisites.md)
- [rollback-drill.md](./rollback-drill.md) (first STOP)
- [disaster-day.md](../../../disaster-day.md)
