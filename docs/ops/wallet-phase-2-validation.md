# Phase 2 Validation Report — Wallet Limited Traffic

Operational gate between **Cutover PR4 (Limited Traffic)** and **PR5 (Default WalletAddress)**.

This is **not** a new ADR, RFC, or capability. It is the ADR 018 Phase 2 evidence pack:
fill the tables from **staging** (or production Limited Traffic window), then record a
recommendation. **PR5 must not start until Recommendation = Proceed to Phase 3.**

| Field | Value |
|-------|-------|
| Status | **ACTIVE** (awaiting KPI evidence) |
| ADR | [018 — Wallet Product Activation](../adr/018-wallet-product-activation-strategy.md) |
| Capability | [#116 Wallet Cutover](https://github.com/roamkit-net/roamkit-docs/issues/116) |
| Code (PR4) | [roamkit-api#64](https://github.com/roamkit-net/roamkit-api/pull/64) |
| Environment | staging (preferred) / noted below |
| Window start (UTC) | |
| Window end (UTC) | |
| Cohort size (start) | |
| Report author | |
| Reviewed by (Ops) | |
| Reviewed by (Product) | |

## Current cutover decision (governance)

```text
PR4: ACCEPTED
Phase 2: ACTIVE
PR5: NOT YET APPROVED
```

## How to collect KPIs (api host)

```bash
# Cutover + shadow + cohort counters
python manage.py wallet_metrics

# Ops status (observation / convert backlog)
python manage.py wallet_ops_status

# Data Migration Validation (if re-check needed)
python manage.py wallet_backfill_addresses --validate --sample-size=20
```

Key fields from `wallet_metrics`:

| KPI | Metric key |
|-----|------------|
| Shadow match rate | `shadow_match_rate` |
| Shadow match / mismatch | `shadow_match_total` / `shadow_mismatch_total` |
| Critical divergences | `shadow_critical_total` |
| Cohort size | `cutover_cohort_size` |
| Cohort completed deposits | `cutover_cohort_deposits_completed` |
| Conversion success (non-shadow credited) | `credited_count` |
| Rollback posture | `cutover_rollback_status` |

Instant rollback drill (no deploy): set `WALLET_CUTOVER_COHORT_ACCOUNT_IDS=` empty →
`cutover_rollback_status=legacy_only`.

---

## KPI results (fill from live window)

| KPI | Target | Result | PASS? |
|-----|--------|--------|:-----:|
| `shadow_match_rate` | ≈ 100% (known Warning exceptions classified) | | |
| Critical divergence | **0** | | |
| Duplicate credits | **0** | | |
| Rollback drill (empty cohort → legacy) | **PASS** | | |
| Backfill validation | **PASS** (no critical errors) | | |

### Evidence links

| Artifact | Link / note |
|----------|-------------|
| `wallet_metrics` export (dated) | |
| Shadow Decision sample / Critical=0 note | |
| Ledger check: no dual `deposit:` + `wallet-obs:` credit | |
| Empty-cohort rollback drill note | |
| `wallet_backfill_addresses --validate` output | |

---

## Checklist (PASS / FAIL)

| Check | PASS? | Notes |
|-------|:-----:|-------|
| Shadow | | Critical = 0 for GO window; Warnings classified |
| Cohort | | Allowlist-only; deposits complete via Wallet path |
| Rollback | | Empty allowlist restores ADR 010 without deploy |
| Metrics | | match_rate, cohort size, deposits, rollback status observed |
| Support | | Briefed on WalletAddress + L1–L3 + legacy shared policy |

---

## Stop criteria (any Critical → remain in Phase 2)

Do **not** recommend Phase 3 / PR5 if:

- Critical shadow divergence > 0
- Duplicate credit detected
- Shadow / limited path grants unexpected production Credits outside Conversion V2
- Empty-cohort rollback fails
- Backfill validation has unresolved critical errors

---

## Recommendation

```text
[ ] Proceed to Phase 3  →  GO PR5
[ ] Remain in Phase 2
```

**Decision (date UTC):**  

**Signed (Ops / Product):**  

### Notes

-

---

## PR5 scope (locked — do not expand)

When Recommendation = Proceed to Phase 3, PR5 may **only**:

| Allowed | Forbidden |
|---------|-----------|
| Remove cohort requirement for default path | Shared wallet retirement |
| `WalletAddress` becomes default for new deposits | Deleting shared wallet logic |
| Shared wallet → legacy mode (still watchable) | Changing Observation semantics |
| | New architecture / new features |

Normative: ADR 018 Phase 3. Review question remains: **consistent with ADR 017 + ADR 018?**

## Related

- [Wallet Product Activation](./wallet-product-activation.md)
- [Evidence of Gate](./evidence-of-gate.md)
- [Wallet Operations](./wallet-operations.md)
- [ADR 018](../adr/018-wallet-product-activation-strategy.md)
