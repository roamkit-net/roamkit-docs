# Evidence — Prepare rollback retry prerequisites

Operational task only. **Does not** run the rollback drill. **Does not** change Criterion #4 (remains **RED** until a PASS drill). Gate C unchanged.

| Field | Value |
|-------|-------|
| Prepared at (UTC) | 2026-07-26T11:20:00Z (approx; host writes immediate) |
| Host | `/opt/stacks/roamkit-production/` |
| Stack mutated? | **No** running containers / `.env` image tags unchanged |
| Host files written | `rollback-target.env`, `.previous-tag` (→ N−1), `.previous-tag.bak-pre-retry-prereq-*` |
| Prior attempt | [rollback-drill.md](./rollback-drill.md) — STOP at Go/No-Go |

## DoD

| # | Requirement | Status |
|---|-------------|:------:|
| 1 | Valid API + WEB N−1 in GHCR, same migration head as N, SHAs recorded | ✅ |
| 2 | Traefik / procedure aligned (no plan vs reality gap) | ✅ |
| 3 | Go/No-Go checks can pass for next drill attempt | ✅ |

---

## 1. Rollback targets (N and N−1)

| Role | API | WEB |
|------|-----|-----|
| **N** (running / restore target) | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` | `7546edb2f732a5e2de46cb35b0dab3bc66993f6c` |
| **N−1** (rollback target) | `86ab449e6975b81a448cd0149fec6bd9a85b0662` | `bf62035325279c87153a93d71efc167a9bc5f80d` |

Full image refs:

```text
N_API_IMAGE=ghcr.io/roamkit-net/roamkit-api:caa3f1d0e3e4c53bacf76719a0f18b1473560207
N_WEB_IMAGE=ghcr.io/roamkit-net/roamkit-web:7546edb2f732a5e2de46cb35b0dab3bc66993f6c
N1_API_IMAGE=ghcr.io/roamkit-net/roamkit-api:86ab449e6975b81a448cd0149fec6bd9a85b0662
N1_WEB_IMAGE=ghcr.io/roamkit-net/roamkit-web:bf62035325279c87153a93d71efc167a9bc5f80d
```

### Migration head

Compared `find …/migrations/*.py | sort | sha256sum` inside images:

| Image | Migration files hash |
|-------|----------------------|
| N API `caa3f1d…` | `9cd77fa6ed35a0917b340f77bdd825508d294b51988d8b89d42be0f6cf0673bc` |
| N−1 API `86ab449…` | `9cd77fa6ed35a0917b340f77bdd825508d294b51988d8b89d42be0f6cf0673bc` |

**Identical.** Tip apps: `orders.0003_order_idempotency_key`, `billing.0001_billing_schema`, `catalog.0004_location_coverages`, `esims.0002_topup_model`.

N−1 API is parent of Gate C `/version` commit (`86ab449` → `caa3f1d`); safe image rollback without migrate-down.

### GHCR

- `docker pull` N−1 API → OK  
- `docker pull` N−1 WEB → OK  

### Host source of truth for next drill

File on host (not in git): `/opt/stacks/roamkit-production/rollback-target.env`  
`.previous-tag` rewritten to N−1 pair so `rollback-production.sh` has a **distinct** API+WEB target. Running `.env` still points at N.

---

## 2. Traefik / procedure alignment

### Reality (2026-07-26)

- Live containers: `traefik.enable=true`
- Public `https://api.roamkit.net/version` serves production N
- Dress-rehearsal assumption (`traefik.enable=false`) is **obsolete** for this stack

### Decision (locked for Criterion #4 retry)

**Mode: live Traefik** — drill runs against the production stack while Traefik serves public Hosts.

Do **not** flip `traefik.enable=false` for this retry (would drop public apex/API). Isolation mode remains documented for future non-public stacks only.

### Safeguards (required before first rollback command)

| Safeguard | Requirement |
|-----------|-------------|
| Change Freeze | No `main` merges; no other prod deploys; no ad hoc `.env` / GHCR tag edits for drill images |
| Window | Operator-approved short window; expect brief public blips on api/web during image swap |
| Owner | Facilitator + Approval recorded in drill evidence |
| Backup | Confirm `roamkit_production_*.sql.gz` present before Go/No-Go |
| Abort | Same Abort Conditions as plan; on FAIL restore N via `deploy-production.sh` with N tags from `rollback-target.env` |
| Verify | After rollback **and** after restore: localhost smoke **and** public `https://api.roamkit.net/version` Expected vs Observed |
| Scope | Criterion #4 only — no Observability / Billing E2E / Gate D / Airalo work in the same window |

### Go/No-Go Traefik check (replaces blind `traefik.enable=false`)

| Mode | Check that must be ✅ |
|------|------------------------|
| **live** (this retry) | Documented mode = live; `traefik.enable=true` on running api/web; public `/version` reachable; safeguards above acknowledged |
| **isolated** (future only) | Documented mode = isolated; `traefik.enable=false`; public Hosts not served by this stack |

---

## 3. Go/No-Go preview (ready for retry)

| Check | Result | Notes |
|-------|:------:|-------|
| Distinct N / N−1 image | ✅ | API and WEB both differ |
| Migration head identical | ✅ | hash `9cd77fa6…` |
| DB backup exists | ✅ | `/var/backups/roamkit_production_20260725T*.sql.gz` |
| `.previous-tag` valid (targets N−1) | ✅ | API+WEB N−1 refs |
| No concurrent deploy | ✅ | Re-check at drill start |
| Traefik check (live mode) | ✅ | Procedure aligned; re-confirm labels + public `/version` at drill start |

**Prerequisite task:** GREEN.  
**Criterion #4:** still RED until drill PASS.

---

## Follow-ups (out of this task)

- Optional post–Criterion #4: allow `ROLLBACK_API_IMAGE` / `ROLLBACK_WEB_IMAGE` or read `rollback-target.env` in `rollback-production.sh` so `.previous-tag` is not the sole SoT.
- Do not use newer API images (e.g. Wave 1 `49607bb…`) as N−1 for current N — migration set may diverge.

## Related

- [rollback-drill.md](./rollback-drill.md) (aborted attempt)
- [disaster-day.md](../../../disaster-day.md)
- Host: `rollback-target.env`, `.previous-tag`
