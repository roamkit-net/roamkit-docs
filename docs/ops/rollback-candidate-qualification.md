# Rollback candidate qualification

Permanent rule for Phase 4 Exit Criterion #4 (and later production rollback drills).

> A rollback target must support the **same verification procedure** as the current release — not only the same migration head.

## Qualification checklist (N−1 and N)

Before declaring any release pair valid for a rollback drill, **both** N and N−1 must satisfy:

| Check | Required |
|-------|----------|
| Distinct API + WEB image SHAs (N ≠ N−1 for both) | ✅ |
| Identical Django migration head (file set hash / tip) | ✅ |
| Same smoke contract as `smoke-test-production.sh` expects | ✅ |
| Same health contract (`/health/live`, `/health/ready`, **`/version`** with non-empty `git_sha`, plus any other smoke hard-fails) | ✅ |
| Images pullable from GHCR | ✅ |
| SHAs documented (`rollback-target.env` and/or evidence pack) | ✅ |

Any ❌ → candidate is **not** a valid N−1 / N for the drill. Fix targets before GO.

## Why

- Attempt 1 ([rollback-drill.md](./releases/1.0.0/evidence/rollback-drill.md)): infra / Traefik / distinct-image blockers.
- Attempt 2 ([rollback-drill-retry.md](./releases/1.0.0/evidence/rollback-drill-retry.md)): N−1 `86ab449` shared migration head but **lacked `/version`**, so smoke DoD was unwinnable on live Traefik.

Migration parity alone is insufficient.

## Go/No-Go

This checklist is part of Go/No-Go for Criterion #4. See also live Traefik safeguards in [rollback-retry-prerequisites.md](./releases/1.0.0/evidence/rollback-retry-prerequisites.md).

## Proposed next targets (not yet deployed)

Documented intent only — **Criterion #4 remains RED** until a PASS drill. Establishing N requires an approved deploy under Change Freeze.

| Role | API SHA | WEB SHA | Notes |
|------|---------|---------|-------|
| **N−1** | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` | `7546edb2f732a5e2de46cb35b0dab3bc66993f6c` | Current running release; first API with `/version` |
| **N** (candidate) | `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` | `7064a9b953ad167e7a23dca6f454ca66b09482de` | Same migration hash as N−1; `/version` present; distinct WEB from N−1 |

Alternate API candidates (same mig + `/version`): `fe598aba…`, `d76512e6…`, `cee960c6…`. Alternate WEB: `e30fd810…` (Wave 1 — larger surface; prefer `7064a9b` for first PASS attempt).

## Related

- [disaster-day.md](./disaster-day.md)
- Plan: `.cursor/plans/roamkit.plan.md` — Exit Criterion #4
