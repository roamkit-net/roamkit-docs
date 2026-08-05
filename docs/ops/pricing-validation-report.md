# Pricing Validation Report (ADR 019)

**Official decision record** that the account-pricing backend engine was verified on
staging **before** public API / web / admin surface (PR4–PR6).

Decision Outcome = **PASS** unlocks PR4–PR6. Formal sign-off is below.

| Field | Value |
|-------|-------|
| Report status | **PASS — READY FOR SURFACE** |
| Checklist | [Pricing Validation Checklist](./pricing-validation-checklist.md) |
| ADR | [019 — Account pricing profiles](../adr/019-account-pricing-profiles.md) |
| Environment | staging (`api.staging.roamkit.net`) |
| API image | `ghcr.io/roamkit-net/roamkit-api:764cb1ae88ab5a1c6c1cda293a5e2e5e4770bf03` (PR3) |
| Package | `discover-in-180days-10gb-px` — `L=57.00`, `N=50.00`, Family 5% → `C=54.15` |
| Window start (UTC) | 2026-08-05T22:02:03Z |
| Window end (UTC) | 2026-08-05T22:03:05Z |
| Operator | Auto (staging evidence run) |
| Test account | `pricing-val-20260805220203@example.com` / `account_id=5cc46dd3-cd33-4488-84bd-9e3ebd0bc19a` |
| Technical review | ChatGPT (OpenAI) — evidence summary assessment |
| Signed off by | Ante Vrcan |
| Evidence pack location | [releases/pricing-adr019-validation/evidence/](./releases/pricing-adr019-validation/evidence/) |

## Governance

```text
Architecture: LOCKED
ADR 019: ACCEPTED
PR1–PR3: MERGED + staging image 764cb1a
Checklist: EXECUTED
This Report: PASS — READY FOR SURFACE
PR4–PR6: UNLOCKED
```

---

## Results table

| Report # | Scenario | Checklist # | Result | Evidence | Notes |
|----------|----------|-------------|--------|----------|-------|
| R1 | Legacy (Flag OFF) | 4 | **PASS** | order `188`; ledger `18a845e6-…feb1`; charge `57.00`; flag=false; `2026-08-05T22:02:22Z` | Charged `L`; fingerprint unused (`-|-|-`) |
| R2 | Family 5% purchase | 1 | **PASS** | order `189`; slug `family` v1; hash `2e385d16…cdc3`; fp `3af82ac5…\|1\|2e385d16…`; ledger `9c782f75-…7c22`; charge `54.15`; flag=true | `money_round(57×0.95)=54.15` |
| R3 | Refund uses snapshot | 2 | **PASS** | same order `189`; `refund_ledger_id=aa5f7785-…ed49b`; refund `+54.15` | Ops credit from `Order.retail_price_usd` |
| R4 | Profile changed → refund | 2 | **PASS** | profile → 20% v2; refund still `54.15`; live resolve hyp `50.00` (wholesale floor) | No re-resolve; hyp ≠ snapshot |
| R5 | Archived profile → refund | 3 | **PASS** | order `190`; archive `2026-08-05T22:02:46Z`; refund `b4711d44-…cb993` `+54.15`; hyp after archive `57.00` | Snapshot independent of profile state |
| R6 | Replay / retry | 6 | **PASS** | order `191`; debit_count `1→1`; same_order; fp unchanged; key `pricing-r6-1785967368-10955` | Airalo 429 on first fulfill → compensate from snapshot; replay no second debit |
| R7 | Deterministic preview | 7 | **BLOCKED** until PR4 | — | Complete in post-PR4 addendum |

### Expected vs Actual (summary)

| # | Expected | Actual |
|---|----------|--------|
| R1 | Charge `L=57.00` with flag OFF | `57.00` debit; flag false |
| R2 | Charge `C=54.15` with Family 5% | `54.15`; profile snapshot present |
| R3 | Refund = snapshotted `C` | `+54.15` |
| R4 | After 20% edit, refund still `C` (not live resolve) | refund `54.15`; live hyp `50.00` |
| R5 | After archive, refund still snapshotted charge | refund `54.15`; archive recorded |
| R6 | Same order, one debit, same fingerprint | PASS (see `R6.json` retry_logs) |

### Per-scenario evidence (minimal fields)

**R1**

| Field | Value |
|-------|-------|
| order_id | `188` |
| account_id | `5cc46dd3-cd33-4488-84bd-9e3ebd0bc19a` |
| pricing_profile_slug / version | *(empty / null — flag OFF)* |
| quote_fingerprint | `-|-|-` |
| pricing_context_hash | *(empty)* |
| ledger_entry_id | `18a845e6-1846-4c6d-9e5d-6386e343feb1` |
| charged amount | `57.00` |
| timestamp | `2026-08-05T22:02:22.476728+00:00` |
| PRICING_PROFILES_ENABLED | `false` |

**R2**

| Field | Value |
|-------|-------|
| order_id | `189` |
| account_id | `5cc46dd3-cd33-4488-84bd-9e3ebd0bc19a` |
| pricing_profile_slug | `family` |
| pricing_profile_version | `1` |
| quote_fingerprint | `3af82ac5-cd50-4c96-83ef-5502fca1715d\|1\|2e385d1657fe279c57623d426b929f166a0f2c4ad20764c4cc95e3fd4f89cdc3` |
| pricing_context_hash | `2e385d1657fe279c57623d426b929f166a0f2c4ad20764c4cc95e3fd4f89cdc3` |
| ledger_entry_id | `9c782f75-4d17-4adf-8b44-62d07bd97c22` |
| charged amount | `54.15` |
| timestamp | `2026-08-05T22:02:39.933871+00:00` |
| PRICING_PROFILES_ENABLED | `true` |

**R3 / R4** (same purchase `189`)

| Field | Value |
|-------|-------|
| order_id | `189` |
| refund_ledger_id | `aa5f7785-d809-409b-903b-6c63947ed49b` |
| refund amount | `54.150000` |
| snapshot_used | `54.15` |
| profile after edit | `family` discount `20.00`, version `2` |
| hypothetical_live_resolve | `50.00` (≠ snapshot) |
| PRICING_PROFILES_ENABLED | `true` |

**R5**

| Field | Value |
|-------|-------|
| order_id | `190` |
| charged / refund | `54.15` / `+54.150000` |
| refund_ledger_id | `b4711d44-ca4a-4b17-b64e-5ff5eaccb993` |
| archived_at | `2026-08-05T22:02:46.042419+00:00` |
| hypothetical_live_resolve after archive | `57.00` (≠ snapshot) |

**R6**

| Field | Value |
|-------|-------|
| order_id | `191` |
| account_id | `5cc46dd3-cd33-4488-84bd-9e3ebd0bc19a` |
| pricing_profile_slug / version | `family` / `1` |
| quote_fingerprint | `5b677c6b-0cf7-4f03-a0bc-8fe675d3a136\|1\|139feec863ce06d414dab9314366ec0e46945387ce3fd4a28345294e42ad3894` |
| pricing_context_hash | `139feec863ce06d414dab9314366ec0e46945387ce3fd4a28345294e42ad3894` |
| ledger_entry_id | `1a697fb5-2247-482c-a358-279c680f3727` (single debit) |
| charged amount | `54.15` |
| retry_logs | `idempotency_key=pricing-r6-1785967368-10955 replay; debit_count 1->1; same_order=True` |
| PRICING_PROFILES_ENABLED | `true` |
| Notes | First fulfill hit Airalo `429`; compensate refunded snapshot `54.15`; replay returned same order |

Raw JSON: `docs/ops/releases/pricing-adr019-validation/evidence/*.json` (+ `pack.json`).

---

## Technical review

```text
Technical review: ChatGPT (OpenAI)

Assessment:
Based on the evidence summary provided in the staging validation, scenarios R1–R6
satisfy the documented Expected outcomes. No inconsistencies were identified in the
supplied evidence summary.

Recommendation:
READY FOR SURFACE
```

Note: this review is based on the operator evidence summary shared in chat. It is
**not** a substitute for formal sign-off by a person who inspected the attached
artifacts and holds release authority.

## Decision (formal)

```text
Capability: Account Pricing Profiles

Validation: PASS

Architecture:
✓ Locked
✓ Implemented
✓ Validated

Evidence:
Attached — docs/ops/releases/pricing-adr019-validation/evidence/

Signed off by:
Ante Vrcan

Date:
2026-08-05

Decision:
PASS — READY FOR SURFACE

Unlocked:
- PR4 (API + internal preview + leak guard)
- PR5 (Web dual price)
- PR6 (Admin preview)

Blocked until later:
- Scenario R7 / checklist #7 (preview determinism) — addendum after PR4
- Production enable of PRICING_PROFILES_ENABLED for real cohorts — separate ops decision
```

| Field | Value |
|-------|-------|
| Outcome | `PASS` |
| Engine status | **READY FOR SURFACE** |
| Date (UTC) | 2026-08-05 |
| Signed off by | Ante Vrcan |
| Technical review | ChatGPT (OpenAI) — summary assessment; GO recommendation |

---

## Post-PR4 addendum (Scenario R7)

After PR4 merges, append:

| Report # | Result | Evidence | Notes |
|----------|--------|----------|-------|
| R7 Preview == purchase | | Preview response + Order snapshot | Same Account + package → identical customer amount |

---

## Related

- [Pricing Validation Checklist](./pricing-validation-checklist.md) (execution gate)
- [ADR 019](../adr/019-account-pricing-profiles.md)
- Analogous pattern: [Wallet Phase 2 Validation Report](./wallet-phase-2-validation.md)
