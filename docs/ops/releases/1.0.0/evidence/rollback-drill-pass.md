# Evidence — Phase 4 Exit Criterion #4 (Rollback drill) — PASS

> Additional verification of Exit Criterion #4 only. Does **not** reopen Gate C GO (2026-07-25).

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Attempt | GO #2 (after GO #1 N establish) |
| Verdict | **PASS** |
| Criterion #4 | **RED → GREEN** |
| Traefik mode | live |
| Closed at (UTC) | 2026-07-26T11:33:11Z |
| Prior attempts | [rollback-drill.md STOP#1](./rollback-drill.md) superseded by this PASS file naming — see History; [retry STOP](./rollback-drill-retry.md); [prereqs](./rollback-retry-prerequisites.md); [N established](./rollback-n-established.md) |

## Rollback Owner

```text
Facilitator: Engineering (solo operator / agent execution)
Observer: —
Approval: Operator GO #2 (2026-07-26)
```

## Targets

| Role | API SHA | WEB SHA |
|------|---------|---------|
| N (restore) | `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` | `7064a9b953ad167e7a23dca6f454ca66b09482de` |
| N−1 (rollback) | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` | `7546edb2f732a5e2de46cb35b0dab3bc66993f6c` |

## Go / No-Go (GO #2 start)

| Check | Result |
|-------|:------:|
| Distinct N / N−1 | ✅ |
| Migration head identical | ✅ |
| Candidate qualification (`/version` both) | ✅ |
| DB backup | ✅ `/var/backups/roamkit/` |
| `.previous-tag` → N−1 | ✅ |
| No concurrent deploy | ✅ |
| Traefik live + public `/version` | ✅ |
| Running = N | ✅ |

## Expected vs Observed SHA

| Moment | Expected API | Expected WEB | Observed API | Observed WEB | Result |
|--------|--------------|--------------|--------------|--------------|:------:|
| After rollback | `caa3f1d…` | `7546edb…` | `caa3f1d…` (local+public `/version`) | image `7546edb…` | **PASS** |
| After restore | `9989fe…` | `7064a9b…` | `9989fe…` (local+public `/version`) | image `7064a9b…` | **PASS** |

## Timestamps + Recovery Time

```text
rollback start UTC:  2026-07-26T11:32:20Z
rollback finish UTC: 2026-07-26T11:32:33Z
restore start UTC:   2026-07-26T11:32:56Z
restore finish UTC:  2026-07-26T11:33:11Z

Recovery Time = rollback finish − rollback start = 13s
Target: <10 min
Actual: 13s (PASS vs target)
```

## Command summary

1. `./scripts/rollback-production.sh` → N−1 images up + health.
2. Align `.env` `API_IMAGE`/`WEB_IMAGE`/`ROAMKIT_*` to N−1 + recreate (required: `env_file` overrides image bake for `/version`).
3. `./scripts/smoke-test-production.sh` → **PASSED** on N−1.
4. Set `.env` to N → `./scripts/deploy-production.sh` → **Deploy successful** + smoke PASSED.
5. Re-confirm `/version` + smoke on N → **PASSED**.

## Smoke

| Phase | Result |
|-------|--------|
| Post-rollback | `PRODUCTION SMOKE PASSED (dress-rehearsal)` |
| Post-restore | `PRODUCTION SMOKE PASSED (dress-rehearsal)` (+ deploy embedded smoke) |

Host smoke script is compose-exec dress-rehearsal variant; public `/version` checked separately both phases.

## Lessons Learned

```text
What worked:
- Qualified N/N−1 pair with shared migration + /version contract.
- Live Traefik drill with Change Freeze completed without Abort.
- Recovery Time well under 10 min.

What should improve:
- deploy/rollback scripts should sync ROAMKIT_* (or stop loading them from .env) so /version always matches the running image tag without a manual recreate step.
- Prefer promoting host smoke-test-production.sh to the full public HTTPS script post–Gate D.

Actions:
- Optional follow-up (not Criterion #4): fix ROAMKIT_* override in compose/deploy path.
- Proceed to Exit Criterion #2 (Observability) when ready — separate step.
```

## History (audit trail)

1. STOP — infra / Traefik / distinct image ([first attempt evidence retained in git history / PR #31](./rollback-drill.md) if present on branch; this file is the PASS record).
2. STOP — N−1 lacked `/version` ([rollback-drill-retry.md](./rollback-drill-retry.md)).
3. Prereqs + qualification rule ([rollback-retry-prerequisites.md](./rollback-retry-prerequisites.md), [rollback-candidate-qualification.md](../../../rollback-candidate-qualification.md)).
4. GO #1 establish N ([rollback-n-established.md](./rollback-n-established.md)).
5. **GO #2 PASS** (this document).

## Related

- [disaster-day.md](../../../disaster-day.md)
- [gate-c.md](./gate-c.md) (unchanged GO)
