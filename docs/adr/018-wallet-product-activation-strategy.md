# ADR 018: Wallet Product Activation Strategy

| Field | Value |
|-------|-------|
| Status | **Proposed** |
| Date | 2026-08 |
| Deciders | Product / Engineering / Ops (pending Ops Review + Product GO) |
| Architecture Review | **PASS** (2026-08) |
| Ops Review | **Pending** |
| Product GO | **Pending** |
| Depends on | [ADR 017](./017-roamkit-wallet-platform.md) (Accepted), [ADR 010](./010-polygon-usdt-prepaid-credits.md) (Accepted) |
| Index | [Wallet Architecture Index](../architecture/wallet-architecture-index.md) |
| Ops | [Wallet Product Activation](../ops/wallet-product-activation.md) |

## ADR Scope

> **ADR 018 decides how RoamKit safely activates the Wallet Platform (ADR 017) as the production deposit intake path, replacing the shared ADR 010 platform wallet for new deposits. It is a release / activation decision — not a new domain architecture.**

This ADR is the **last architectural-operational document** before production cutover. It does **not**:

- invent new Wallet domain entities or reopen RFC 003–006,
- amend ADR 017 standing rules or Credits/`CreditService` authority,
- introduce Wallet v2, multi-chain, or ADR 019,
- imply that merge of Wallet Platform capabilities alone cut over production.

After this ADR is Accepted, further expansion of **this** document’s scope is forbidden. Implementation proceeds only via capability **Wallet Cutover (ADR010 Migration)** under milestone **Wallet Product v1**.

## Context

**Wallet Platform v1** (capabilities #92–#98) is complete on `develop`: allocation, observation, confirmation, credit conversion, funding adapters, ops drills, metrics.

Production money path **today** remains ADR 010:

```text
deposit-info → POLYGON_PLATFORM_WALLET → TXID verify → CreditService → Credits
```

**Target** money path (ADR 017):

```text
deposit-info → WalletAddress(Account) → Observation → Confirmed → Credit Conversion → Credits
```

ADR 017 Exit Criteria already required an explicit follow-on for this cutover. This ADR is that follow-on.

## Decision

RoamKit activates Wallet Product via a **phased, flag-gated, dual-path (shadow) cutover** with explicit data-migration validation, divergence severity, rollback levels, legacy watch, and retirement review — before shared-wallet retirement.

### ADR 010 Supersession Policy

Which ADR is authoritative for **deposit intake** in each phase. Credits ledger / `CreditService` rules from ADR 010 (and ADR 012) remain binding for money mutations forever; only the **intake path** is superseded.

| Phase | ADR 010 (deposit intake) | ADR 018 |
|-------|--------------------------|---------|
| Phase 0–2 | **Primary** | Proposed / shadowing |
| Phase 3 | Shared (legacy for non-default path) | **Primary** for new deposits |
| Phase 4 | Legacy only (watch shared address) | **Primary** |
| Phase 5 | **Superseded** for new deposits | **Primary** |

There is no implicit “ADR 018 always wins.” Reviewers use this table.

### Data Migration Gate (before Phase 1 Shadow)

| Account | WalletIdentity | WalletAddress | Action |
|---------|----------------|---------------|--------|
| New user | Create on demand | Allocate | OK |
| Existing without wallet | **Backfill** | Allocate | Required before Shadow |
| Existing with wallet | No-op | Existing | OK |

**Gate exit**

- Every Account that may fund has a `WalletIdentity`.
- Backfill policy documented (batch pre-allocate vs lazy allocate for cohort).
- Dry-run backfill report reviewed.

**Validation (required — migration is not “deployed” until this passes)**

- Sampled verification of backfilled Identity / Address rows.
- Orphan `WalletIdentity` count = 0 (or explicitly explained and accepted).
- Duplicate **active** `WalletAddress` per Identity + Chain = 0.
- Missing Index Registry fields (`derivation_index`, `address`) = 0.

Gate **fails** if validation fails. Do not enter Shadow.

### Feature flags

Env-style flags (same pattern as existing `*_ENABLED` settings). Default **off** in production until gates pass.

| Flag | Role |
|------|------|
| `WALLET_ADDRESS_ENABLED` | Allocate / expose per-Account `WalletAddress` |
| `OBSERVATION_ENABLED` | Run Observation ingest / confirmation path |
| `CREDIT_CONVERSION_V2` | Convert Confirmed Observations via Cap 3 path |
| `SHADOW_MODE` | Dual-path compare without making Wallet the sole intake |

Phase → flag matrix lives in [ops/wallet-product-activation.md](../ops/wallet-product-activation.md).

### Cutover phases

```text
Phase 0 Preparation
    ↓
Phase 1 Shadow
    ↓
Phase 2 Limited Traffic
    ↓
Phase 3 Default WalletAddress
    ↓
Phase 4 Shared Wallet Legacy
    ↓
Phase 5 Shared Wallet Retirement
```

Shared wallet does **not** disappear on the same day as Phase 3.

### WalletAddress Immutability Rule

Once a `WalletAddress` is exposed through `/deposit-info` (or equivalent funding UX), it **must never silently change**.

Rotation = allocate a **new** address → previous → **retired** (still watchable) → never replace-in-place. Bookmarked addresses must remain a valid receive path for the watch window.

### Shadow success criteria and divergence classification

**GO metrics (examples; thresholds may be refined in ops without amending architecture)**

- 100% of new funding intents receive a `WalletAddress` when `WALLET_ADDRESS_ENABLED` is on.
- Shadow Observation covers the same deposit set as the legacy verify path.
- Conversion amount / Observation Identity match expectations.
- **Duplicate credit = 0**.
- Observation lag / convert latency within stated SLO bounds.

**Not every mismatch stops rollout.**

| Divergence | Severity | Action |
|------------|----------|--------|
| Missing Observation | **Critical** | Stop advancement; consider rollback |
| Duplicate Credit | **Critical** | Stop; incident; never silent ledger rewrite |
| Observation latency | **Warning** | Continue with monitoring; remediate SLO |
| Metadata mismatch (non-identity) | **Warning** | Continue; track on cutover dashboard |

**Stop criteria:** any **Critical** divergence (or Warning rate above ops-defined threshold).

### Rollback levels

Prefer L1 → L2 → L3. Never delete or rewrite ledger history.

| Level | Action |
|-------|--------|
| **L1** | Disable WalletAddress allocation (`WALLET_ADDRESS_ENABLED=false`) |
| **L2** | `deposit-info` returns shared platform wallet again |
| **L3** | Disable Observation / Conversion v2; resume ADR 010 verify flow |

### Legacy Shared Wallet Policy and Retirement Review

- After Phase 4: continue watching the shared address for an **ops-set** window (suggested default: **6 months** after Phase 3 default flip); user messaging as needed.
- **Legacy Retirement Review** (gate before Phase 5 OFF): remaining inbounds on shared address? bookmark / support-ticket evidence? residual risk acceptable?
- Only then Phase 5 retirement. Late deposits after retirement → incident + Billing remediation (no silent credit from explorer alone).

### Cutover audit events

Every phase transition and rollback emits a durable audit record. Examples:

```text
CutoverPhaseChanged
Phase3Activated
RollbackLevel2
LegacyRetired
```

Ops must reconstruct which phase / flags were live at any time.

### Cutover dashboard

Ops surface (not Credits SoT): allocations/day, observation lag, conversion latency, shadow divergences by severity, legacy shared deposits, flag / rollback status, last audit event. May start as CLI/`wallet_metrics` + cutover counters; UI later.

### Wallet Readiness Gate

Required before Phase 2+ advancement and before **Phase 3 GO**:

| Area | Must show |
|------|-----------|
| Infrastructure | Allocation, Observation, Conversion operable |
| Operations | Metrics, alerting, recovery drill, dashboard |
| Product | Add Funds shows `WalletAddress` when intended; Funding Provider; explorer |
| Rollback | L1–L3 documented and drillable |
| **Support Readiness** | Support knows WalletAddress, rollback levels, legacy policy |

**Support Readiness is mandatory before Phase 3.**

### GO Authority

| Gate | Owner |
|------|-------|
| Architecture Review | Architecture |
| Ops Review | Operations |
| Product GO | Product Owner |

**Phase 3 activation requires all three approvals.** No single role may authorize default WalletAddress intake alone.

### Evidence of Gate (Wallet Readiness)

Checkboxes alone are insufficient. Each Readiness item must retain **evidence** (link, report, screenshot, or run ID) under ops notes / Evidence of Gate pack:

| Check | Evidence (examples) |
|-------|---------------------|
| Shadow Critical mismatch = 0 | Dashboard / metrics export for the GO window |
| Rollback drill | Runbook reference + dated drill note |
| Data Migration Validation | Validation report (dry-run + sampled checks) |
| Flags | Deployment / env evidence for intended flag matrix |
| Support Readiness | Briefing note or checklist sign-off |

### Production Freeze (pre–Phase 3)

After a successful Wallet Readiness Gate and until Phase 3 activation completes (or is explicitly aborted):

> **No unrelated Wallet changes may be deployed between successful Readiness Gate and Phase 3 activation.**

Cutover-only changes (flags, monitoring, documented hotfixes required for the gate) are allowed. Unrelated Wallet / billing intake features wait.

### Post-Cutover Review (mandatory after Phase 3)

Operational release reviews at **24 h**, **72 h**, and **7 days** after Phase 3 activation. At each checkpoint review at least:

- Critical / Warning shadow or production divergences
- Duplicate credits (must remain 0)
- Support tickets related to deposit / address / funding
- Rollback decision (hold / L1–L3 / continue)

Record outcomes with audit events / ops notes. This is release hypercare, not a new architecture track.

## Consequences

**Positive**

- Explicit authority table for ADR 010 vs ADR 018 by phase.
- Safer activation via shadow, flags, and graded rollback.
- Clear freeze: no ADR 019 / Wallet v2 / multi-chain in this cycle.
- Named GO owners and evidence requirements before Phase 3.

**Negative / cost**

- Dual-path and legacy watch operational burden for a defined window.
- Cutover capability implementation work after Accept.
- Production freeze window constrains unrelated Wallet deploys.

**Neutral**

- ADR 017 remains Wallet constitution; this ADR only activates it.

## ADR Acceptance Criteria

ADR 018 becomes **Accepted** only when **all** of the following hold:

1. Architecture Review **PASS** — **done** (2026-08)
2. Ops Review **PASS** — **pending**
3. Product **GO** — **pending**

Until then status remains **Proposed**. **No Cutover implementation before Accept.** Merge of this document as Proposed does not authorize Phase 3.

## Binding after Accept

- Cutover PRs: **Is this consistent with ADR 017 + ADR 018?**
- Capability work only under milestone **Wallet Product v1**.
- **Do not expand this ADR’s scope** after Accept (or after this pre-Accept ops-gates amendment); new product ideas wait for a future cycle.

## Explicit non-goals (locked)

- ADR 019
- Wallet Platform / Product v2
- Multi-chain beyond Polygon-first already locked in ADR 017

## Related

- [ADR 017 — RoamKit Wallet Platform](./017-roamkit-wallet-platform.md)
- [ADR 010 — Polygon USDT prepaid credits](./010-polygon-usdt-prepaid-credits.md)
- [ADR 012 — Billing extensibility](./012-billing-extensibility-rules.md)
- [Wallet Product Activation (ops)](../ops/wallet-product-activation.md)
- [Evidence of Gate](../ops/evidence-of-gate.md)
- [Wallet Operations](../ops/wallet-operations.md)
- [Wallet Architecture Index](../architecture/wallet-architecture-index.md)
- [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md)
