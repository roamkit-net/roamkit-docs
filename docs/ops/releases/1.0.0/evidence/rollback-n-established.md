# Evidence — Establish rollback-qualified N (GO #1)

**Scope:** deploy new **N** only. **Rollback drill not run** (awaits separate GO #2).  
Criterion #4 remains **RED**. Gate C unchanged.

| Field | Value |
|-------|-------|
| Prepared / deployed (UTC) | 2026-07-26T11:28:50Z → 11:29:05Z (deploy); acceptance 11:29:56Z |
| Traefik mode | live |
| Host | `/opt/stacks/roamkit-production/` |
| Rollback executed? | **No** |

## Targets (established)

| Role | API | WEB |
|------|-----|-----|
| **N** (running) | `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` | `7064a9b953ad167e7a23dca6f454ca66b09482de` |
| **N−1** (`.previous-tag`) | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` | `7546edb2f732a5e2de46cb35b0dab3bc66993f6c` |

Host SoT: `rollback-target.env`, `.previous-tag`, `.env` (`API_IMAGE`/`WEB_IMAGE` + `ROAMKIT_*` aligned to N).

## Acceptance (before GO #2)

| Check | Result | Evidence |
|-------|:------:|----------|
| `/version` on **N** (localhost + public) | ✅ | `git_sha=9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| `/version` route exists on **N−1** image | ✅ | `path("version")` in `caa3f1d` urls |
| Migration head identical | ✅ | hash `9cd77fa6…` both |
| `smoke-test-production.sh` on N | ✅ | PASSED (dress-rehearsal compose exec on host) |
| Running image tags = planned N | ✅ | compose ps: api `9989fe…`, web `7064a9b…` |
| No further deploys after verification | ✅ | STOP after acceptance |
| Candidate qualification | ✅ | See [rollback-candidate-qualification.md](../../../rollback-candidate-qualification.md) |

## Notes

- First `deploy-production.sh` succeeded, but host `.env` still had `ROAMKIT_GIT_SHA=caa3f1d…`, which **overrides** image bake via `env_file`. Fixed by aligning `ROAMKIT_*` to N and `--force-recreate` (no second image pull of a different SHA).
- Migrations: `No migrations to apply`.
- Public `https://roamkit.net/` → HTTP 200 after web healthy.

## STOP

**Do not run** `rollback-production.sh` until explicit **GO #2**.

## Related

- [rollback-candidate-qualification.md](../../../rollback-candidate-qualification.md)
- [rollback-drill-retry.md](./rollback-drill-retry.md)
- [rollback-retry-prerequisites.md](./rollback-retry-prerequisites.md)
