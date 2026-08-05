# Phase 2 Validation Report — Wallet Limited Traffic

Operational gate between **Cutover PR4 (Limited Traffic)** and **PR5 (Default WalletAddress)**.

This is **not** a new ADR, RFC, or capability. It is the ADR 018 Phase 2 evidence pack:
fill **Result** from **staging** Limited Traffic, apply the GO rule below, then record
**Decision**. Code review is closed for this gate — only operational evidence counts.

**PR5 is BLOCKED until Decision Outcome = Proceed to Phase 3.**

| Field | Value |
|-------|-------|
| Report status | **AWAITING RESULTS** |
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

## Governance status

```text
Architecture: COMPLETE
Platform: COMPLETE

PR4: ACCEPTED

Phase 2: ACTIVE / RUNNING
Phase 2 Validation Report: AWAITING RESULTS
Evidence: PENDING

PR5: BLOCKED pending Phase 2 PASS
     (NOT AUTHORIZED until Decision = Proceed to Phase 3)
```

## What is in scope now

| Look at | Do not look at |
|---------|----------------|
| Staging Limited Traffic results | Code / new PRs |
| KPI **Result** column | “feels fine” |
| Formal Decision block | Subjective exceptions |

---

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

Report is ready for Decision **only when every Result cell is filled**.

| KPI | Target | Result | Status |
|-----|--------|--------|:------:|
| `shadow_match_rate` | ≈ 100% (or documented ops threshold; Warnings classified) | | PASS / FAIL |
| Critical divergence | **0** | | PASS / FAIL |
| Duplicate credits | **0** | | PASS / FAIL |
| Rollback drill (empty cohort → legacy) | **PASS** | | PASS / FAIL |
| Backfill validation | **PASS** (no critical errors) | | PASS / FAIL |

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

| Check | Status | Notes |
|-------|:------:|-------|
| Shadow | | Critical = 0 for GO window; Warnings classified |
| Cohort | | Allowlist-only; deposits complete via Wallet path |
| Rollback | | Empty allowlist restores ADR 010 without deploy |
| Metrics | | match_rate, cohort size, deposits, rollback status observed |
| Support | | Briefed on WalletAddress + L1–L3 + legacy shared policy |

---

## GO rule (non-subjective)

```text
If every Gate criterion above is PASS
        ↓
Recommendation / Decision Outcome:
Proceed to Phase 3

Else
        ↓
Recommendation / Decision Outcome:
Remain in Phase 2
```

No exceptions. No “looks good enough.”

### Stop criteria (any → Remain in Phase 2)

- Critical shadow divergence > 0
- Duplicate credit detected
- Shadow / limited path grants unexpected production Credits outside Conversion V2
- Empty-cohort rollback fails
- Backfill validation has unresolved critical errors

---

## Recommendation (derived from GO rule)

```text
[ ] Proceed to Phase 3  →  unlocks GO PR5
[ ] Remain in Phase 2
```

---

## Decision

Final audit record for why Phase 2 closed (or stayed open).

```text
Decision

Date (UTC):
Owner (Ops / Product):

Outcome:

☐ Proceed to Phase 3
☐ Remain in Phase 2

Reason:
```

After Outcome = Proceed to Phase 3, set report header Status to **PASS** and unlock PR5.
After Outcome = Remain in Phase 2, keep Status **AWAITING RESULTS** or set **FAIL — remain Phase 2** and continue the window.

---

## PR5 scope (locked — do not expand)

When Decision Outcome = Proceed to Phase 3, PR5 may **only**:

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
