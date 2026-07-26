# Evidence — Phase 4 Exit Criterion #1 (Production stack / Gate D) — execution

| Field | Value |
|-------|-------|
| Criterion | Phase 4 Exit #1 — Production stack / Gate D |
| Inventory | [gate-d-criterion-1-inventory.md](./gate-d-criterion-1-inventory.md) (merged #39) |
| Execute GO | **Conditional** (2026-07-26) — Closure plan 1–7 under STOP rules |
| Execution recorded (UTC) | 2026-07-26T12:05:00Z |
| Verdict (this pack) | **EXECUTE PASS** (steps 1–7) · Hypercare **COMPLETE** |
| Criterion #1 status | 🟢 **GREEN** — Hypercare closed 2026-07-26T22:38:24Z without unresolved P0/P1 |
| Gate D close-out | **GO** — [gate-d.md](./gate-d.md) |
| Gate C | **GO** (unchanged) |

## STOP checks (all clear)

| STOP | Result |
|------|--------|
| Web remains baked on `api.staging.roamkit.net` | ❌ did not trigger — running digest bakes `api.roamkit.net` only |
| Catalog sticky skeleton | ❌ did not trigger — Playwright happy path **PASS** |
| `main` deploy pulls unpaired cross-repo SHA | ❌ avoided — **did not** create/promote `roamkit-api` `main` (CI would set `WEB_IMAGE=…api:${api_sha}`) |

## What ran

### 1–2. `roamkit-web` `main` + production CI bake

| Step | Detail |
|------|--------|
| Bootstrap | Created `origin/main` at running SHA `7064a9b953ad167e7a23dca6f454ca66b09482de` |
| Promote PR | [roamkit-web#28](https://github.com/roamkit-net/roamkit-web/pull/28) `develop` → `main` squash-merged → `8afd7268f55ecc63b1684010582521da036be4fd` |
| CI bake @ `7064a9b…` | [run 30201062127](https://github.com/roamkit-net/roamkit-web/actions/runs/30201062127) — Assert baked API host **OK** (`require=api.roamkit.net`, `forbid=api.staging…`) |
| CI bake @ `8afd726…` | [run 30201231530](https://github.com/roamkit-net/roamkit-web/actions/runs/30201231530) — same assert **OK** |
| `roamkit-api` `main` | **Not created** this window (cross-repo same-SHA `deploy-production` STOP risk + develop tip includes Faza 5 API beyond current prod N) |

### 3. Production deploy (web only)

| Field | Value |
|-------|-------|
| Stack | `/opt/stacks/roamkit-production/` |
| API (unchanged N) | `ghcr.io/roamkit-net/roamkit-api:9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| WEB (redeployed) | `ghcr.io/roamkit-net/roamkit-web:7064a9b953ad167e7a23dca6f454ca66b09482de` |
| WEB digest | `sha256:732856f00bb3bafe00f03e0bfead5892081759d5d226871665c2ae3fc3eeaa09` (main rebuild; was `c169bcaa…` staging bake) |
| Method | `docker compose --profile app up -d --no-deps --force-recreate web` |
| Health | `healthy` |

Running container bake check: `FORBID_OK` + `REQUIRE_OK`.

### 4. Public catalog verification

```text
PLAYWRIGHT_BASE_URL=https://roamkit.net npx playwright test e2e/catalog-price.spec.ts --grep "happy path"
→ 1 passed (~2s): prices visible, catalog-price-skeleton count = 0
```

Re-confirmed after `main` tip bake published.

Public chunk serves `api.roamkit.net` (no staging host).

### 5. Public smoke

| URL | HTTP |
|-----|------|
| `https://api.roamkit.net/health/live` | 200 |
| `https://api.roamkit.net/health/ready` | 200 |
| `https://api.roamkit.net/version` | 200 (`git_sha=9989fe…`) |
| `https://api.roamkit.net/api/v1/billing/config/` | 200 |
| `https://roamkit.net/` | 200 |
| `https://roamkit.net/global-esim` | 200 |

Host `smoke-test-production.sh` remains dress-rehearsal compose-exec (backlog to restore public HTTPS script; not required after public curls + Playwright).

### 6. Docs / decision (this PR)

- Manifest filled for current N
- Capability + go-live checklist catalog/bake boxes updated
- Release Decision Log: Gate D **GO WITH CONDITIONS** (hypercare clock)

### 7. Hypercare (closed)

| Field | Value |
|-------|-------|
| Started | `2026-07-25T22:38:24Z` |
| Ended (24h) | `2026-07-26T22:38:24Z` |
| Samples | 96 · `HYPERCARE_COMPLETE` · 2 transient ALERTs (recovered) · no unresolved P0/P1 |
| Rollback window | **COMPLETE** `2026-07-25T22:37:56Z` |
| Close-out | [gate-d.md](./gate-d.md) Success Snapshot + Decision Log **GO** |

## Post-execution inventory

| # | Requirement | Status |
|---|-------------|:------:|
| 1 | Public DNS | 🟢 |
| 2 | Traefik Hosts | 🟢 |
| 3 | Cutover + rollback window | 🟢 |
| 4 | Hypercare ≥24h | 🟢 |
| 5 | Production web bake | 🟢 |
| 6 | Web `main` promotion path | 🟢 (`roamkit-api` `main` deferred — see STOP) |
| 7 | `/global-esim` real prices | 🟢 |
| 8 | Public smoke (curls + Playwright) | 🟢 |
| 9 | Flags / migrations | 🟢 |
| 10 | Manifest filled | 🟢 (this PR) |
| 11 | Decision log Gate D | 🟢 **GO** |
| 12 | Capability / checklist | 🟢 |
| 13 | Gate D unconditional exit | 🟢 [gate-d.md](./gate-d.md) |

## Out of scope / backlog

- Create `roamkit-api` `main` only after fixing cross-repo image pairing in `deploy-production.yml`
- Promote/deploy web `8afd726…` (wizard) when API N includes lifecycle contract
- Restore host `smoke-test-production.sh` to public HTTPS
- Traefik SNI / Kuma public HTTPS retarget
- Airalo / Phase 5

## Related

- [gate-d-cutover.md](../../../gate-d-cutover.md)
- [gate-d-criterion-1-inventory.md](./gate-d-criterion-1-inventory.md)
- ADR 013
