# Wallet Product Activation

Ops companion to [ADR 018 — Wallet Product Activation Strategy](../adr/018-wallet-product-activation-strategy.md) (**Accepted**).

Implements:

```text
ADR 018
Capability: Wallet Cutover (docs)
```

This is **release engineering**, not new Wallet architecture. ADR 017 remains the constitution.

## ADR 010 vs ADR 018 (deposit intake)

| Phase | ADR 010 | ADR 018 |
|-------|---------|---------|
| Phase 0–2 | **Primary** | Accepted (shadowing) |
| Phase 3 | Shared (legacy) | **Primary** for new deposits |
| Phase 4 | Legacy only | **Primary** |
| Phase 5 | **Superseded** for new deposits | **Primary** |

Credits / `CreditService` rules stay under ADR 010 + ADR 012 always.

## Feature flag matrix (indicative)

| Phase | WALLET_ADDRESS_ENABLED | OBSERVATION_ENABLED | CREDIT_CONVERSION_V2 | SHADOW_MODE |
|-------|:----------------------:|:-------------------:|:--------------------:|:-----------:|
| 0 Preparation | off | off | off | off |
| 1 Shadow | on (cohort) | on | off or dry | **on** |
| 2 Limited traffic | on (limited) | on | on (limited) | on or taper |
| 3 Default WalletAddress | **on** | **on** | **on** | off (or residual) |
| 4 Shared legacy | on | on | on | off |
| 5 Shared retired | on | on | on | off |

Exact cohort definitions are ops parameters; do not invent new domain rules.

## Wallet Readiness Gate (before Phase 3 GO)

### Infrastructure

- [ ] Allocation works
- [ ] Observation works
- [ ] Credit Conversion works
- [ ] Data Migration Gate **and Validation** passed

### Operations

- [ ] Metrics / cutover dashboard fields available
- [ ] Alerting for Critical divergences
- [ ] Recovery drill passed ([wallet-operations.md](./wallet-operations.md))
- [ ] Rollback L1–L3 documented and drillable

### Product

- [ ] Add Funds shows `WalletAddress` when intended
- [ ] Funding Provider path works
- [ ] Explorer / attribution path understood

### Support Readiness (mandatory before Phase 3)

- [ ] Support knows what a `WalletAddress` is
- [ ] Support knows rollback levels L1–L3
- [ ] Support knows legacy shared-wallet policy

### Rollback

- [ ] L1 allocation off
- [ ] L2 `deposit-info` → shared wallet
- [ ] L3 Observation / Conversion v2 off → ADR 010 verify

## GO Authority

| Gate | Owner |
|------|-------|
| Architecture Review | Architecture |
| Ops Review | Operations |
| Product GO | Product Owner |

**Phase 3 activation requires all three approvals.**

Current ADR 018 review status: Architecture **PASS** · Ops **PASS** · Product **APPROVED** · ADR **Accepted**.

Cutover capability [#116](https://github.com/roamkit-net/roamkit-docs/issues/116) is **in implementation** (small PRs).

```text
PR1–PR4: done (flags → backfill → shadow → limited traffic cohort)
Phase 2: ACTIVE — fill [Phase 2 Validation Report](./wallet-phase-2-validation.md)
PR5 (Default WalletAddress): NOT YET APPROVED until Phase 2 Recommendation = Proceed
```

Phase 3 still needs live Readiness Gate evidence + all three GO Authority approvals at activation time.

## Evidence of Gate (required artifacts)

| Check | Evidence |
|-------|----------|
| Shadow Critical mismatch = 0 | Dashboard / metrics export |
| Rollback drill | Runbook ref + dated note ([wallet-operations.md](./wallet-operations.md)) |
| Data Migration Validation | Validation report |
| Flags | Deploy / env evidence |
| Support Readiness | Briefing / sign-off |
| **Phase 2 Limited Traffic** | [Phase 2 Validation Report](./wallet-phase-2-validation.md) (KPI table + Recommendation) |

File under Evidence of Gate / release notes when closing the readiness window. See [evidence-of-gate.md](./evidence-of-gate.md).

## Phase 2 Validation Gate (before PR5)

Limited Traffic is an **operational gate**, not a formality. Do not start Cutover PR5 until the [Phase 2 Validation Report](./wallet-phase-2-validation.md) records **Proceed to Phase 3**.

Required KPI evidence (staging window): `shadow_match_rate`, Critical divergence = 0, duplicate credits = 0, empty-cohort rollback PASS, backfill validation PASS.

## Production Freeze (pre–Phase 3)

After successful Readiness Gate until Phase 3 activation completes or aborts: **no unrelated Wallet deploys**. Cutover-only changes allowed.

## Post-Cutover Review

After Phase 3, mandatory checkpoints at **24 h**, **72 h**, and **7 days**:

- [ ] Divergences (Critical / Warning)
- [ ] Duplicate credits = 0
- [ ] Support tickets (deposit / address / funding)
- [ ] Rollback decision recorded

## Data Migration Validation checklist

- [ ] Dry-run backfill report reviewed
- [ ] Sampled row verification OK
- [ ] Orphan WalletIdentity = 0 (or accepted exception)
- [ ] Duplicate active WalletAddress per identity+chain = 0
- [ ] Missing Index Registry fields = 0

## Shadow divergence cheat sheet

| Divergence | Severity | Action |
|------------|----------|--------|
| Missing Observation | Critical | Stop; consider rollback |
| Duplicate Credit | Critical | Stop; incident |
| Observation latency | Warning | Monitor; remediate |
| Metadata mismatch | Warning | Track on dashboard |

## Phase checklist

- [ ] Phase 0 Preparation (flags off; migration gate)
- [ ] Phase 1 Shadow (Critical = 0 for GO window)
- [ ] Phase 2 Limited traffic + **[Phase 2 Validation Report](./wallet-phase-2-validation.md) PASS**
- [ ] Phase 3 Default WalletAddress (**Readiness Gate + Support** — only after Phase 2 Proceed)
- [ ] Phase 4 Shared wallet legacy watch
- [ ] **Legacy Retirement Review** then Phase 5 retirement

### Legacy Retirement Review (before Phase 5 OFF)

- [ ] Remaining inbounds on shared address?
- [ ] Bookmark / support-ticket evidence?
- [ ] Residual risk accepted by Product + Ops?

## Audit events (minimum set)

```text
CutoverPhaseChanged
Phase3Activated
RollbackLevel1 | RollbackLevel2 | RollbackLevel3
LegacyRetired
```

## Cutover dashboard fields

- Wallet allocations / day
- Observation lag
- Conversion latency
- Shadow divergences (by severity)
- Legacy shared deposits
- Flag / rollback status
- Last cutover audit event

## Immutability reminder

Once exposed via `/deposit-info`, a `WalletAddress` must not silently change. Rotate = new address + retire old.

## Related

- [ADR 018](../adr/018-wallet-product-activation-strategy.md)
- [ADR 017](../adr/017-roamkit-wallet-platform.md)
- [Phase 2 Validation Report](./wallet-phase-2-validation.md)
- [Wallet Operations](./wallet-operations.md)
- [Launch Gates](./launch-gates.md) (general; this doc is Wallet-specific activation)
- [SLO targets](./slo.md)
