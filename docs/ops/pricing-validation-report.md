# Pricing Validation Report (ADR 019)

**Official decision record** that the account-pricing backend engine was verified on
staging **before** public API / web / admin surface (PR4–PR6).

Fill this file (or a dated copy under `docs/ops/releases/<release>/evidence/`) **after**
staging execution of
[pricing-validation-checklist.md](./pricing-validation-checklist.md).
Do **not** open PR4 until Decision Outcome = **PASS** and evidence is attached.

| Field | Value |
|-------|-------|
| Report status | **TEMPLATE — AWAITING STAGING** |
| Checklist | [Pricing Validation Checklist](./pricing-validation-checklist.md) |
| ADR | [019 — Account pricing profiles](../adr/019-account-pricing-profiles.md) |
| Environment | staging |
| Window start (UTC) | |
| Window end (UTC) | |
| Operator | |
| Reviewed by | |
| Evidence pack location | |

## Governance (before fill)

```text
Architecture: LOCKED
ADR 019: ACCEPTED
PR1–PR3: MERGED
Checklist: READY FOR STAGING
This Report: AWAITING RESULTS
PR4–PR6: BLOCKED
```

## Recommended staging order

Execute in this order so a failure points at a layer:

1. Flag OFF (legacy) → same as production charge path  
2. Flag ON + Family 5% → discount + snapshot + ledger  
3. Refund → snapshot amount  
4. Profile changed after purchase → refund unchanged  
5. Profile archived → refund / replay still work  
6. Replay → same fingerprint + charged amount  
7. Preview → **after PR4** only  

---

## Results table

Map report rows to checklist scenarios. Fill after staging; leave blank until then.

| Report # | Scenario | Checklist # | Result | Evidence | Notes |
|----------|----------|-------------|--------|----------|-------|
| R1 | Legacy (Flag OFF) | 4 | | `order_id`, `ledger_id`, timestamp, flag=OFF | Legacy charge verified |
| R2 | Family 5% purchase | 1 | | `order_id`, fingerprint, snapshot, ledger | Discount applied correctly |
| R3 | Refund uses snapshot | 2 or 3 (refund path) | | `refund_ledger_id`, original `C` | Snapshot amount used |
| R4 | Profile changed → refund | 2 | | Original charge + refund; profile version after edit | No re-resolve |
| R5 | Archived profile → refund | 3 | | Order snapshot + refund; `archived_at` | Snapshot independent of live profile |
| R6 | Replay / retry | 6 | | Same `order_id`, idempotency key, single debit | Same fingerprint + charged amount |
| R7 | Deterministic preview | 7 | **BLOCKED** until PR4 | Preview JSON + purchase snapshot | Complete in post-PR4 addendum |

### Per-PASS artifacts (minimum)

For each **PASS** row, evidence must include (or link to):

- `order_id` (or `topup_id`)
- `account_id`
- `pricing_profile_slug` / `pricing_profile_version` (when flag ON)
- `pricing_context_hash` (and quote fingerprint if recorded)
- `ledger_entry_id` (+ refund id when applicable)
- charged amount (`retail_price_usd` / Topup `amount`)
- timestamp (UTC)
- `PRICING_PROFILES_ENABLED` value for that run

---

## Decision — PASS template

Use when R1–R6 are all **PASS** and evidence is attached (R7 may remain BLOCKED).

```text
Capability: Account Pricing Profiles

Validation: PASS

Architecture:
✓ Locked
✓ Implemented
✓ Validated

Evidence:
Attached (see Evidence pack location / table above)

Decision:
READY FOR SURFACE

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
| Date (UTC) | |
| Signed off by | |

---

## Decision — FAIL template

Use when any of R1–R6 is **FAIL**. Do **not** edit the checklist definition; record failure here.

```text
Capability: Account Pricing Profiles

Validation: FAIL

Blocked:
- PR4
- PR5
- PR6

Failed scenario(s):
- R# …

Root Cause:
…

Required Fix:
… (backend / config / ops — not a checklist rewrite)

Re-validation:
Scenarios R#–R# (re-run after fix; attach new evidence)

Decision:
NOT READY FOR SURFACE
```

| Field | Value |
|-------|-------|
| Outcome | `FAIL` |
| Engine status | **NOT READY FOR SURFACE** |
| Date (UTC) | |
| Signed off by | |
| Tracking issue / PR for fix | |

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
