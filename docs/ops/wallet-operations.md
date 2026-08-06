# Wallet Operations — Failure Domains & Recovery

Ops runbook for [ADR 017](../adr/017-roamkit-wallet-platform.md) **Failure Domains
and Expected Recovery**. Effects are **delays / unavailability** — never silent
ledger corruption.

Capability: [Wallet Operations (#97)](https://github.com/roamkit-net/roamkit-docs/issues/97).

Implements:

```text
ADR 017
Capability: Wallet Operations
```

## Standing rules (ops)

1. Credits mutations only via `CreditService` (never rewrite ledger rows).
2. Credit Conversion Trigger = **Confirmed Observation** only — Funding Provider
   status / explorer / RPC alone never grants Credits.
3. Recovery for addresses = **Seed + Index Registry** (indices never reused).
4. Do not invent a side ledger when Billing is down.

## Failure Domains → recovery

| Failure | Effect | Expected recovery | Drill / command |
|---------|--------|-------------------|-----------------|
| RPC / observation adapter unavailable | Observation delayed; Pending Confirmation stalls; **no** false Credited | Resume ingest when adapter recovers; backfill/replay via Observation Identity (`chain + tx_hash + log_index`) | Re-ingest same identity (idempotent); `wallet_ops_status` |
| Indexer / explorer unavailable | Same | Same; **do not** credit from explorer alone | Same |
| Funding Provider unavailable | New funding UX delayed | Existing `WalletAddress` + Credits unaffected; resume when provider recovers | Adapter `status()` / `metadata()` only — not Credits SoT |
| Wallet DB unavailable | Allocation / Index Registry / new Observations blocked | Resume allocation + attribution after DB recovery | `/health/ready`; then `wallet_ops_status` |
| Platform Wallet Infrastructure (seed) unavailable | New address materialization / sweep blocked | Watch may continue if DB + adapters up; resume when seed/runtime recovers | Confirm `WALLET_HD_MNEMONIC` in secured env (never in git) |
| Billing / `CreditService` unavailable | Credits delayed; Confirmed Observations wait | Resume convert when Billing recovers — **no** side ledger | `wallet_resume_converts --dry-run` then `--apply` |
| Chain reorg after Credited | Incident + Billing remediation | Never silent rewrite of ledger history; compensating entries only via Billing rules | Incident tree below + `billing_reconcile_balances` |

## First-response trees

### Tree: Pending Confirmation stuck

```text
Observation stuck in pending_confirmation
        │
        ▼
wallet_ops_status  (counts + oldest pending)
        │
        ├── RPC down → wait / switch RPC; do not Confirmed from explorer alone
        │
        └── RPC OK → re-ingest signal with higher confirmations
                (DepositObservationService.ingest — idempotent)
                │
                ├── still pending → depth < Chain Policy; wait
                │
                └── confirmed → Cap 3 convert (wallet_resume_converts)
```

### Tree: Confirmed but not Credited

```text
Observation confirmed (or conversion_started) — balance unchanged
        │
        ▼
wallet_resume_converts --dry-run
        │
        ├── Billing / DB down → wait; do not invent Credits
        │
        └── OK → wallet_resume_converts --apply
                (CreditConversionService; idempotent on Observation Identity)
                │
                └── still wrong → CreditService / ledger incident;
                    billing_reconcile_balances (report only)
```

### Tree: Funding UX broken, Credits fine

```text
MEXC / Binance / on-ramp unavailable
        │
        ▼
Existing WalletAddresses + Credits unchanged
        │
        ▼
Pause funding UX; do not treat provider hisrec as SoT
        │
        ▼
Resume FundingProvider.deposit() when provider recovers
```

### Tree: Reorg after Credited

```text
Chain reorg invalidates a Credited Observation
        │
        ▼
P0 incident — do NOT delete / rewrite ledger rows
        │
        ▼
Document Observation Identity + ledger entry
        │
        ▼
Compensating CreditService debit (human-approved) if product policy requires
        │
        ▼
billing_reconcile_balances; post-incident note
```

## Drill checklist (ops)

Run periodically (can fold into [Disaster Day](./disaster-day.md)):

- [ ] `wallet_ops_status` — healthy counts; no unexpected `conversion_started` backlog
- [ ] Dry-run `wallet_resume_converts --dry-run` — empty or understood queue
- [ ] Simulate Pending → re-ingest with depth ≥ `POLYGON_MIN_CONFIRMATIONS` → Confirmed
- [ ] Simulate Confirmed → `--apply` convert → Credited once; second apply idempotent
- [ ] Funding provider `metadata()` / `status()` reachable without touching Credits
- [ ] Confirm seed present only in secured env (`WALLET_HD_MNEMONIC`)

Record date + operator under `releases/` or a dated note when run in production.

## Commands (`roamkit-api`)

| Command | Purpose |
|---------|---------|
| `wallet_ops_status` | Observation status counts, convert backlog, seed configured flag |
| `wallet_resume_converts` | Resume Confirmed / Conversion Started → Credited (`--dry-run` / `--apply`) |
| `wallet_metrics` | Allocation / observation / confirmation / convert counters (not Credits SoT) |

Examples:

```bash
python manage.py wallet_ops_status
python manage.py wallet_metrics
python manage.py wallet_resume_converts --dry-run
python manage.py wallet_resume_converts --apply --limit 50
```

## Related

- [ADR 018](../adr/018-wallet-product-activation-strategy.md) — Wallet Product Activation (Proposed)
- [Wallet Product Activation (ops)](./wallet-product-activation.md)
- [Incident runbook](./incident-runbook.md)
- [Disaster Day](./disaster-day.md)
- [Billing dashboard](./billing-dashboard.md)
