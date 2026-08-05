# ADR 017: RoamKit Wallet Platform (architecture adoption)

| Field | Value |
|-------|-------|
| Status | **Proposed** |
| Date | 2026-08 |
| Deciders | Product / Engineering (post RFC 003–006 Architecture Reviews) |
| Index | [Wallet Architecture Index](../architecture/wallet-architecture-index.md) |

## ADR Scope

> **ADR 017 ratifies the frozen Wallet architecture defined by Vision, Conversion Boundary, and RFC 003–006. It does not introduce new architecture.**

This ADR is the **constitution for Wallet implementation**: it makes the frozen design binding for capabilities and PRs, states how Wallet relates to Billing (ADR 010/012), and records operational failure/recovery expectations.

It does **not**:

- invent new domain entities or money rules beyond the frozen RFCs + ADR 010/012,
- cut over production from the shared ADR 010 deposit address in this document alone,
- open RFC 007 or expand architectural scope for this cycle.

## Context

RoamKit’s production money path today is [ADR 010](./010-polygon-usdt-prepaid-credits.md): shared Polygon platform wallet, exact-amount deposit verification, Credits ledger via `CreditService`.

A full Wallet architecture was designed as proposals:

```text
Vision → Conversion Boundary → RFC 003 → RFC 004 → RFC 005 → RFC 006
```

Those RFCs are Architecture Review Passed / Frozen. Without a Wallet ADR, implementers lack a single **normative** “this is what we build” decision, and ADR 010 remains the only Accepted money path.

Canonical money path (target):

```text
Funding
    ↓
WalletAddress
    ↓
Confirmed Observation
    ↓
Credit Conversion
    ↓
Credits Ledger
```

Separation of responsibilities: blockchain is not business logic; Credits are not blockchain; Funding Provider is not Wallet; Wallet is not Billing.

## Decision

Adopt the **RoamKit Wallet Platform** target architecture as ratified from the frozen RFC set. Implementation and cutover from ADR 010’s shared `POLYGON_PLATFORM_WALLET` require follow-on capability PRs and may use additional ADRs for Chain Policy numbers, schema, and rollout — they must not contradict this ADR or ADR 010/012 Billing invariants.

### Normative Sources

| Topic | Source | Notes |
|-------|--------|--------|
| Wallet Domain | [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) | Account → WalletIdentity → WalletAddress → Deposit |
| Wallet Infrastructure | [RFC 004](../rfcs/004-platform-wallet-infrastructure.md) | HD allocation, Index Registry, recovery |
| Funding | [RFC 005](../rfcs/005-funding-provider-interface.md) | Funding Provider adapters → WalletAddress |
| Observation | [RFC 006](../rfcs/006-deposit-observation-confirmation.md) | Observation SM, Identity, Confirmation Policy |
| Billing / Credits | [ADR 010](./010-polygon-usdt-prepaid-credits.md) (+ [ADR 012](./012-billing-extensibility-rules.md)) | Ledger SoT; `CreditService` sole mutator |

Frozen RFCs are **normative for Wallet design**. ADR 010 remains **normative for production Credits** until an Accepted cutover ADR/PR says otherwise.

### Non-Normative References

| Document | Purpose |
|----------|---------|
| [Wallet Vision](../architecture/roamkit-wallet-platform-vision.md) | Strategic direction (non-normative) |
| [Conversion Boundary](../architecture/wallet-conversion-boundary.md) | Why chain stops at Credits (explanatory) |
| [Architecture Freeze](../architecture/wallet-architecture-freeze.md) | Change-control process for frozen docs |
| [Architecture Index](../architecture/wallet-architecture-index.md) | Navigation map |
| [Sandbox framework](../architecture/wallet-sandbox.md) / [Exit Artifacts](../architecture/wallet-sandbox-artifacts/) | Research evidence |
| Capability plans / DoDs | Implementation planning (not architecture SoT) |
| [Funding Provider Interface Contract](../architecture/funding-provider-interface-contract.md) | Logical adapter ops appendix |

### Standing rules (locked — from frozen RFCs)

1. Funding Providers never define `WalletIdentity`.
2. Credit Conversion Trigger = **Confirmed Observation** only (not RPC, not Funding Provider, not address allocation alone).
3. Observation Identity = `Chain + TxHash + LogIndex` (or chain-equivalent).
4. Recovery = Seed + Index Registry; indices never reused; allocation idempotent.
5. Credits mutations only via `CreditService`; adapters never write the ledger.
6. Blockchain boundary ends at credit conversion; product ops (including auto renew) use Credits only.

### Implementation Constraints

Every Wallet-related capability and PR **must** obey:

| # | Constraint |
|---|------------|
| 1 | **No bypass of `WalletAddress`** — inbound value attribution is to a RoamKit-owned address (RFC 004), not to a Funding Provider identity or shared undocumented address (except the temporary ADR 010 path until cutover). |
| 2 | **No Credits before Confirmed Observation** — nothing may call `CreditService` for chain deposits until Observation is **Confirmed** (RFC 006). |
| 3 | **Funding Providers remain interchangeable adapters** — no provider hard-codes into Wallet domain or Credits SoT (RFC 005). |
| 4 | **Credits Ledger remains authoritative after conversion** — Wallet state must not mutate or shadow ledger history (RFC 003 / ADR 010). |
| 5 | **Observation Identity idempotency** — duplicate adapter signals collapse to one Observation (`Chain + TxHash + LogIndex`). |
| 6 | **Deviations require a new ADR** — capabilities may not silently reinterpret Normative Sources. |

Prefer **small capabilities** (e.g. Address Allocation, Deposit Observation, Credit Conversion, per-provider Funding adapters), each with its own DoD — not one monolith project.

### Failure Domains and Expected Recovery

Effects are delays / unavailability — **not** silent ledger corruption:

| Failure | Effect | Expected Recovery |
|---------|--------|-------------------|
| RPC / observation adapter unavailable | Observation delayed (Pending Confirmation stalls); no false Credited | Resume observation when adapter recovers; backfill/replay via Observation Identity |
| Indexer / explorer unavailable | Same; fallbacks are ops choice, not alternate Credits SoT | Same; do not credit from explorer alone |
| Funding Provider unavailable | New funding UX delayed | Existing WalletAddresses and Credits unaffected; resume funding when provider recovers |
| Wallet DB unavailable | Address allocation / Index Registry blocked; no new Observations attributed | Resume allocation and attribution after DB recovery |
| Platform Wallet Infrastructure (seed/signing) unavailable | New address materialization / sweep blocked | Watch may continue if DB+adapters up; resume materialization/sweep when infra recovers |
| Billing / `CreditService` unavailable | Credits delayed; Confirmed Observations wait | Resume convert when Billing recovers — **must not** invent a side ledger |
| Chain reorg after Credited | Incident + Billing remediation | Never silent rewrite of ledger history (RFC 006); compensating entries only via Billing rules |

### Relationship to ADR 010

| Today (ADR 010) | Target (this ADR + RFCs) |
|-----------------|--------------------------|
| Shared platform deposit address | Per-Account `WalletAddress` (RFC 004) |
| Verify by tx + exact amount | Observation → Confirmation Policy → Confirmed → convert |
| Credits via `CreditService` | **Unchanged** |
| Polygon USDT first | Unchanged for v1 Chain Policy |

Cutover plan (shared wallet → per-user addresses) is **out of scope for this ADR’s acceptance** as a detailed runbook; it must be an explicit follow-on ADR or implementation plan that preserves ADR 010 balances and ledger history.

### Consequences

**Positive**

- One Accepted (when this ADR is Accepted) implementable Wallet constitution.
- Clear Normative vs Non-Normative sources for engineers and capabilities.
- Failure Domains + recovery expectations before coding.

**Negative / cost**

- Implementation work across Wallet infra, observation, funding adapters, and cutover.
- Temporary dual-path complexity during migration from ADR 010 shared wallet.

**Neutral**

- Vision/RFC documents stay frozen; changes go through research evidence or new ADRs.

## Status

**Proposed** — Architecture Review of RFC 006 passed; this ADR is the acceptance gate before Wallet capabilities.

When **Accepted**:

- Treat Normative Sources, standing rules, and Implementation Constraints as binding for Wallet PRs.
- Do not open new Wallet RFCs for this cycle without freeze-process evidence.
- Prefer small capability/implementation PRs and, if needed, focused ADRs (e.g. Polygon Chain Policy numbers, schema, cutover).

### ADR Exit Criteria

ADR 017 is considered **done as a living constitution** (not “never revisited”) when:

1. Wallet capabilities and related PRs **reference ADR 017** as the binding Wallet decision.
2. The **first** Wallet capability implements against Normative Sources **without deviation**.
3. Any intentional deviation requires a **new ADR** (or Accepted revision of this one) before merge.
4. Production cutover from ADR 010 shared wallet is tracked as an **explicit** follow-on plan/ADR — not implied by accepting this document alone.

Until then, status remains Proposed (or Accepted-with-open-cutover if Accepted before cutover completes).

## Related

- [Wallet Architecture Index](../architecture/wallet-architecture-index.md)
- [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md)
- [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) · [RFC 004](../rfcs/004-platform-wallet-infrastructure.md) · [RFC 005](../rfcs/005-funding-provider-interface.md) · [RFC 006](../rfcs/006-deposit-observation-confirmation.md)
- [ADR 010](./010-polygon-usdt-prepaid-credits.md) · [ADR 012](./012-billing-extensibility-rules.md)
- [Track 1 Exit Artifact](../architecture/wallet-sandbox-artifacts/01-wallet-address-assignment.md)
- [Cross-RFC Consistency Review](../architecture/wallet-sandbox-artifacts/02-cross-rfc-consistency-review.md)
