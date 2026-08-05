# Pricing Validation Checklist (ADR 019)

Operational gate between **backend pricing (PR1–PR3)** and **public surface (PR4–PR6)**.

This is **not** a new ADR. It is the evidence pack that backend charge behaviour matches
[ADR 019](../adr/019-account-pricing-profiles.md) before additive catalog API / web / admin
preview land.

**PR4 (API + preview) is BLOCKED until Decision Outcome = PASS.**

Any FAIL on scenarios **1–6** is a **blocker for PR4** — fix backend / re-run staging
before opening surface PRs. Do not open PR4 while this checklist is only “ready”
or partially filled.

| Field | Value |
|-------|-------|
| Checklist status | **READY FOR STAGING** |
| Report status | **AWAITING RESULTS** |
| Engine status | **NOT READY FOR SURFACE** until Decision Outcome = PASS |
| ADR | [019 — Account pricing profiles](../adr/019-account-pricing-profiles.md) |
| Code (merged) | [roamkit-api#68](https://github.com/roamkit-net/roamkit-api/pull/68) (schema), [#69](https://github.com/roamkit-net/roamkit-api/pull/69) (service), [#70](https://github.com/roamkit-net/roamkit-api/pull/70) (charge path) |
| Environment | staging (required) |
| Window start (UTC) | |
| Window end (UTC) | |
| Operator | |
| Reviewed by | |

## Governance status

```text
Architecture: LOCKED (ADR 019)
PR1 schema: MERGED
PR2 PricingService + snapshots: MERGED
PR3 charge path (debit-from-snapshot): MERGED

Pricing Validation Checklist: READY FOR STAGING
Pricing Validation Report: AWAITING RESULTS
Pricing Engine: NOT READY FOR SURFACE

PR4 (API + preview): BLOCKED pending Report PASS + Evidence attached
PR5 (web dual price): BLOCKED pending PR4
PR6 / PR1a: after surface PRs (or earlier if blocking)
```

## READY FOR SURFACE (formal GO)

Locked rule — do not weaken without an explicit Decision update:

```text
Pricing Engine

Status:
  READY FOR SURFACE

Requirements:
  ✓ PR1 merged (schema)
  ✓ PR2 merged (PricingService + snapshots)
  ✓ PR3 merged (charge path / debit-from-snapshot)
  ✓ Validation checklist scenarios 1–6 PASS
  ✓ Evidence artifacts attached (see below)

Unlocks:
  → PR4 (additive API + internal preview + leak guard)

Does NOT unlock until the above:
  → PR5 (web)
  → Production enable of PRICING_PROFILES_ENABLED for real users
```

Scenario **7** (preview) is an **addendum after PR4**, not a requirement for READY FOR SURFACE.

---

## Required evidence artifacts

Marking a scenario **PASS** without IDs is insufficient. For each of scenarios 1–6,
capture at least:

| Artifact | Where |
|----------|--------|
| `order_id` (or `topup_id`) | Django admin / DB |
| `account_id` | Billing Account UUID |
| `pricing_profile` slug + version (if flag ON) | PricingProfile admin |
| Snapshot charged amount (`retail_price_usd` / Topup `amount`) | Order / Topup row |
| Snapshot fingerprint fields (`pricing_context_hash`, profile id/version when set) | Order / Topup row |
| Ledger entry id(s) + `delta` | `CreditLedgerEntry` |
| Execution timestamp (UTC) | When the scenario was run |
| Flag value (`PRICING_PROFILES_ENABLED`) | Staging env / settings |

Store them in the Evidence column, an appendix under
`docs/ops/releases/<release>/evidence/pricing-validation.md`, or linked internal notes.
The Decision block must reference where the pack lives.

---

## What is in scope now

| Look at | Do not look at |
|---------|----------------|
| Staging purchase / refund / replay with flag | Public API shape (PR4) |
| Order/Topup snapshot fields + ledger deltas | Frontend (PR5) |
| Flag ON vs OFF behaviour | New pricing features |
| Formal Decision block below | “feels correct” without Evidence |

---

## Worked example (amounts)

Use one fixed catalog package for Scenarios 1–6 so numbers are comparable.

| Symbol | Meaning | Example |
|--------|---------|---------|
| `L` | Provider list / MSP (`Package.price_usd`) | `56.00` |
| `N` | Wholesale net (`net_price_usd`) | `50.00` |
| Family | `discount_percent = 5`, `floor_policy = wholesale` | — |
| `C` | Customer charge | `money_round(L × 0.95)` → **`53.20`** (if `C ≥ N`) |

Substitute real staging package IDs/prices in Evidence; Expected column must match
`PricingService.resolve` for that package + profile.

---

## Staging setup (preconditions shared)

1. Staging API deploy includes PR1–PR3 (schema + service + charge path).
2. Django admin: create `PricingProfile` slug `family`, `discount_percent=5.00`,
   `floor_policy=wholesale`, active, effective window covers now.
3. Assign profile to a **test billing Account** (not a random production-like user).
4. Fund the account with enough credits for two purchases + margin.
5. Note package `external_id`, `price_usd` (`L`), `net_price_usd` (`N`).
6. Feature flag env:
   - Scenarios 1–3, 5–6: `PRICING_PROFILES_ENABLED=true`
   - Scenario 4: `PRICING_PROFILES_ENABLED=false` (redeploy or toggle; document which)

Instant rollback (no migration rollback): set `PRICING_PROFILES_ENABLED=false`.

---

## Validation checklist

Fill **Result** (`PASS` / `FAIL` / `BLOCKED`) and **Evidence** (IDs, amounts, links/screenshots)
during staging. Leave empty until executed.

| # | Scenario | Preconditions | Expected | Evidence (collect) | Result |
|---|----------|---------------|----------|--------------------|--------|
| 1 | Family 5% purchase | Flag **ON**; Account has `family` profile; package with known `L` | Charged **`C`** (e.g. `53.20` for `L=56`); Order `list_price_usd=L`, `retail_price_usd=C`; ledger ORDER debit `delta=-C` | Order id; snapshot fields (`list_price_usd`, `retail_price_usd`, `pricing_profile_slug`, `pricing_context_hash`, `snapshot_schema_version`); Ledger entry id + `delta` | |
| 2 | Profile changed after purchase | Scenario 1 Order fulfilled (or failed-after-debit path); then admin changes Family to **20%** (or other) | Refund / compensate equals **original `C`**, not new resolve | Same Order id; profile version after edit; REFUND ledger `delta=+C` (original); prove `retail_price_usd` unchanged on Order | |
| 3 | Profile archived after purchase | Scenario 1 completed; archive Family profile | Refund still succeeds using Order snapshot `C` | Order snapshot; `archived_at` on profile; REFUND ledger `+C` | |
| 4 | Flag OFF → legacy | Flag **OFF**; same Account may still have profile assigned | Charge **`L`** (legacy `package.price_usd`); pricing fingerprint fields empty/unused for charge | Order `retail_price_usd=L`; Ledger debit `-L`; note flag value in Evidence | |
| 5 | Flag ON → discount | Flag **ON**; Family assigned; same package as #1 | Charge **`C`** again (matches #1 math) | Order snapshot + Ledger; compare to Scenario 1 | |
| 6 | Replay / retry | Flag **ON**; trigger retry with **same** `idempotency_key` after first reserve/fulfill (or mid-FULFILLING → complete then retry) | Single debit; same Order; same snapshotted `C`; no second resolve effect | Order id; idempotency key; Ledger count for that `reference_id` (=1 debit); provider call count if observable | |
| 7 | Deterministic preview | **Blocked until PR4** lands preview endpoint | Preview `customer` == later purchase `retail_price_usd` for same Account+package | Preview response JSON; Order snapshot after purchase | **BLOCKED** |

### Scenario notes

- **#2 / #3** prefer a controlled provider-failure path after debit (compensate) *or* an
  admin/ops refund that credits from Order snapshot — whichever staging supports.
  The invariant is: **refund amount = snapshotted charged amount**, never a fresh resolve.
- **#6** directly proves resolve-once: changing the live profile between attempts must not
  change the charged amount on the existing Order.
- **#7** is recorded here so the Validation Report can stay one artifact; execute after PR4.

---

## Pricing Validation Report (fill after staging)

Official decision record: **[pricing-validation-report.md](./pricing-validation-report.md)**.

Copy scenario results into that report (PASS or FAIL template). Attach evidence
under `docs/ops/releases/<release>/evidence/` or link staging admin / ticket IDs.
Do not mark READY FOR SURFACE in the checklist alone — the Report Decision block is authoritative.

Working notes while executing may still use the table below:

| Scenario | Result | Evidence ref |
|----------|--------|--------------|
| 1 Family 5% purchase | PASS | Report R2 → `evidence/R2.json` |
| 2 Profile changed after purchase | PASS | Report R3/R4 → `evidence/R3_R4.json` |
| 3 Profile archived after purchase | PASS | Report R5 → `evidence/R5.json` |
| 4 Flag OFF legacy | PASS | Report R1 → `evidence/R1.json` |
| 5 Flag ON discount | PASS | Report R2 (same math) |
| 6 Replay | PASS | Report R6 → `evidence/R6.json` |
| 7 Deterministic preview | BLOCKED until PR4 | |

Authoritative Results + Decision: **[pricing-validation-report.md](./pricing-validation-report.md)**.

### GO rule

```text
PASS  = Scenarios 1–6 all PASS + evidence artifacts attached for each
FAIL  = any of 1–6 FAIL  →  PR4 remains BLOCKED (treat as backend defect)
BLOCKED = staging unavailable / cannot collect Evidence
```

READY FOR SURFACE is granted **only** when Report Decision Outcome = **PASS** and reviewer signed off.

### Decision

| Field | Value |
|-------|-------|
| Outcome | `PASS` (draft — see Report; awaiting reviewer sign-off) |
| Engine status after decision | `READY FOR SURFACE` after Report sign-off |
| Date (UTC) | 2026-08-05 |
| Signed off by | *(pending — Report)* |
| Evidence pack location | `docs/ops/releases/pricing-adr019-validation/evidence/` |
| Notes | Staging image `764cb1a`; package `discover-in-180days-10gb-px` |
| Unlocks | PR4–PR6 when Report Decision signed PASS |

```text
Decision Outcome: PASS (draft — reviewer sign-off on Report)
Pricing Engine: READY FOR SURFACE (after sign-off)
PR4: NOT AUTHORIZED until Report Signed off by is filled
```
---

## Related

- [ADR 019](../adr/019-account-pricing-profiles.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md) (CreditService / ledger invariants unchanged)
- Analogous gate: [Phase 2 Validation Report (Wallet)](./wallet-phase-2-validation.md)
