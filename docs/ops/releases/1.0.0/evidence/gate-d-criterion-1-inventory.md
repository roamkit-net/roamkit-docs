# Evidence — Phase 4 Exit Criterion #1 (Production stack / Gate D) — inventory

| Field | Value |
|-------|-------|
| Criterion | Phase 4 Exit #1 — Production stack / Gate D |
| Inventory at (UTC) | 2026-07-26T11:52:35Z |
| Gate C | **GO** (unchanged) — [gate-c.md](./gate-c.md) |
| Criteria #4 / #2 / #3 | **GREEN** (do not reopen) |
| Stack | `/opt/stacks/roamkit-production/` |
| Running API | `ghcr.io/roamkit-net/roamkit-api:9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |
| Running WEB | `ghcr.io/roamkit-net/roamkit-web:7064a9b953ad167e7a23dca6f454ca66b09482de` |

Treat this as the **final Phase 4 release gate**, not a routine ops task.
Exit Criterion #1 closes only when Gate D DoD below is **GREEN** with merged evidence.

## Inventory (DoD map)

| # | Requirement | Status | Notes |
|---|-------------|:------:|-------|
| 1 | Public DNS → production | 🟢 GREEN | `api.roamkit.net`, `roamkit.net`, `www.roamkit.net` → 200; staging on `staging.*` |
| 2 | Traefik Host rules on production stack | 🟢 GREEN | `traefik.enable=true`; API `Host(api.roamkit.net)`; web `roamkit.net` \|\| `www.roamkit.net` |
| 3 | Cutover + rollback window already executed | 🟢 GREEN | Cutover `2026-07-25T22:05:27Z`; rollback window **COMPLETE** `22:37:56Z` (no abort) — host logs `gate-d-*` |
| 4 | Hypercare monitor running | 🟡 YELLOW | Started `2026-07-25T22:38:24Z`; ~**13.2 h** elapsed at inventory (~**10.8 h** to 24h). Samples: live/ready/version/web=200, `drift_hits=0` |
| 5 | Production web **bake** = `api.roamkit.net` | 🔴 RED | Image chunks contain **`api.staging.roamkit.net`** only. Runtime `NEXT_PUBLIC_API_URL=https://api.roamkit.net` does **not** rewrite client bundles |
| 6 | GitHub `main` promotion path (ADR 013) | 🔴 RED | Remote branches: **`roamkit-web` / `roamkit-api` = `develop` only** (no `origin/main`). Docker bake uses production URLs **only on `refs/heads/main`**. No production-baked web tag in GHCR yet |
| 7 | `/global-esim` real catalog prices (no sticky skeleton) | 🔴 RED | SSR still emits `catalog-price-skeleton` (×13); depends on #5. `GET /api/v1/billing/config/` on **api.roamkit.net** = 200 (API OK) |
| 8 | Production web smoke (public HTTPS) | 🟡 YELLOW | Host `smoke-test-production.sh` still **dress-rehearsal / compose-exec**. CI deploy smoke hits public API health/version only — no catalog check. Public curls (web/API) succeed |
| 9 | Feature flags / Migration Ready | 🟢 GREEN | Billing ON; WC/subs/vouchers OFF (cutover log). Billing migration applied |
| 10 | Release Manifest filled | 🔴 RED | [manifest.md](../manifest.md) still **DRAFT** (pending prod tags, migrate tip, deploy date, rollback SHA) |
| 11 | Release Decision Log — Gate D row | 🔴 RED | [release-decision-log.md](../../../release-decision-log.md) Gate D still `_pending_` |
| 12 | Capability-status / go-live checklist | 🟡 YELLOW | Platform Observed ⏳; catalog checklist item still blocked on prod bake |
| 13 | Gate D exit = Hypercare 24h + no open P0/P1 | 🟡 YELLOW | Clock running; catalog/#5 gap means cutover acceptance incomplete until bake fix verified |

**Criterion #1 overall (inventory):** 🟡 **YELLOW / ACTIVE** — infrastructure cutover largely done; **production web bake + catalog acceptance + manifest/decision** block GREEN.

## Already true (do not re-cutover blindly)

- Production stack is **live** on public Hosts (not dress-rehearsal isolation).
- Rollback drill (#4), Observability (#2), Billing public DoD (#3) are closed.
- Hypercare daemon is already writing `gate-d-hypercare.log`.

## Closure plan (this criterion only)

1. **Establish `main`** on `roamkit-web` (and `roamkit-api` if deploy pairs SHAs) via PR `develop` → `main` per ADR 013 — **not** force-push leftover local `main`.
2. Let CI build **production-baked** `roamkit-web` (`NEXT_PUBLIC_API_URL=https://api.roamkit.net`); confirm GHCR tag; forbid `api.staging` in image chunks.
3. Deploy that web image to `/opt/stacks/roamkit-production/` (align `WEB_IMAGE` + recreate; keep API N unless promoting API `main` in the same window). Update N−1 pair if needed for rollback contract.
4. Restore / run **public** production smoke; verify `/global-esim` after load: **no** sticky `catalog-price-skeleton`; network host = `api.roamkit.net`.
5. Fill Release Manifest + Gate D evidence pack; update capability-status + checklist.
6. Confirm Hypercare ≥24h from start (or document remediation restart policy if bake deploy resets acceptance clock) with no unresolved P0/P1 → Gate D **GO** in decision log.
7. Only then: Criterion #1 **GREEN** → **Phase 4 = GREEN**.

## Proposed Execute DoD (PASS / STOP)

| Check | PASS |
|-------|------|
| Web image bake | `grep` on `/app/.next/**` shows `api.roamkit.net` and **zero** `api.staging.roamkit.net` |
| Catalog | After load, no sticky `catalog-price-skeleton`; prices visible |
| Smoke | Public HTTPS smoke PASS (API live/ready/version + web origin) |
| Docs | Manifest filled; Gate D evidence + decision log row; capability/checklist updated |
| Hypercare | ≥24h stable **or** explicit GO WITH CONDITIONS if operator accepts remaining clock with catalog PASS already recorded |

**STOP if:** bake still staging; catalog sticky; Traefik/DNS regress; P0/P1 open; `main` promotion would deploy unintended SHA pair without rollback-qualified N−1.

## Out of scope (explicit backlog)

- Traefik SNI / retarget Kuma API monitors to public HTTPS (Criterion #2 backlog)
- Airalo / Phase 5
- Live Polygon TX verify optional backlog
- Vouchers / WC / subscriptions ON
- Full Disaster Day annual drill
- App user notification engine

## Related

- [gate-d-cutover.md](../../../gate-d-cutover.md)
- [launch-gates.md](../../../launch-gates.md)
- [billing-e2e-criterion-3.md](./billing-e2e-criterion-3.md) (catalog bake deferred here)
- ADR 013
