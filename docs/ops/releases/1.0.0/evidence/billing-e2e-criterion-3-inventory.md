# Evidence — Phase 4 Exit Criterion #3 (Billing E2E) — inventory

| Field | Value |
|-------|-------|
| Criterion | Phase 4 Exit #3 — Billing E2E (public) |
| Inventory at (UTC) | 2026-07-26T11:46:00Z |
| Prior Gate C | [billing-e2e.md](./billing-e2e.md) — PASS on localhost dress-rehearsal only |
| Running API SHA | `9989fe559107af3caa5d9b96d73c7dfb95cbfe68` |

## Inventory

| # | Requirement | Status | Notes |
|---|-------------|:------:|-------|
| 1 | Public API reachable (`https://api.roamkit.net`) | 🟢 | `/version`, `/billing/config` 200 |
| 2 | `BILLING_ENABLED=true`, WC off, Polygon env | 🟡 | Confirm in DoD env gate |
| 3 | `production-dod-billing.sh` vs **public** API_URL | 🔴 | Only localhost PASS recorded |
| 4 | Deposit stand-in → ledger → balance → order spend | 🔴 | Pending public run |
| 5 | verify-cex negative path | 🔴 | Pending public run |
| 6 | Catalog `/global-esim` + config on production host | 🟡 | Page 200; price visibility TBD in run |
| 7 | Evidence + checklist SHA/date | 🔴 | Pending PASS |

**Overall:** 🟢 **GREEN** (closed 2026-07-26 — [billing-e2e-criterion-3.md](./billing-e2e-criterion-3.md)). Catalog UI bake → Criterion #1 backlog.


## Out of scope

Gate D · Traefik SNI · Airalo · new billing features · app notifications · Criterion #1.
