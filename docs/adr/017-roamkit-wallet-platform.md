# ADR 017: RoamKit Wallet Platform (architecture adoption)

| Field | Value |
|-------|-------|
| Status | **Proposed** |
| Date | 2026-08 |
| Deciders | Product / Engineering (post RFC 003–006 Architecture Reviews) |
| Index | [Wallet Architecture Index](../architecture/wallet-architecture-index.md) |

## Context

RoamKit’s production money path today is [ADR 010](./010-polygon-usdt-prepaid-credits.md): shared Polygon platform wallet, exact-amount deposit verification, Credits ledger via `CreditService`.

A full Wallet architecture was designed as proposals:

```text
Vision → Conversion Boundary → RFC 003 → RFC 004 → RFC 005 → RFC 006
```

Those RFCs are Architecture Review Passed / Frozen. Without a Wallet ADR, implementers lack a single **normative** “this is what we build” decision, and ADR 010 remains the only Accepted money path.

This ADR does **not** cut over production in one step. It **adopts** the frozen RFC chain as the target Wallet architecture and defines how it relates to Billing, failure domains, and future implementation ADRs/PRs.

## Decision

Adopt the **RoamKit Wallet Platform** target architecture as specified by the frozen RFC set below. Implementation and cutover from ADR 010’s shared `POLYGON_PLATFORM_WALLET` require follow-on PRs and may use additional ADRs for Chain Policy numbers, schema, and rollout — they must not contradict this ADR or ADR 010/012 Billing invariants.

### Normative Sources

| Topic | Source | Notes |
|-------|--------|--------|
| Wallet Domain | [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) | Account → WalletIdentity → WalletAddress → Deposit |
| Wallet Infrastructure | [RFC 004](../rfcs/004-platform-wallet-infrastructure.md) | HD allocation, Index Registry, recovery |
| Funding | [RFC 005](../rfcs/005-funding-provider-interface.md) | Funding Provider adapters → WalletAddress |
| Observation | [RFC 006](../rfcs/006-deposit-observation-confirmation.md) | Observation SM, Identity, Confirmation Policy |
| Billing / Credits | [ADR 010](./010-polygon-usdt-prepaid-credits.md) (+ [ADR 012](./012-billing-extensibility-rules.md)) | Ledger SoT; `CreditService` sole mutator |
| Direction (non-normative) | [Vision](../architecture/roamkit-wallet-platform-vision.md), [Conversion Boundary](../architecture/wallet-conversion-boundary.md) | Why chain stops at Credits |

Frozen RFCs are **normative for Wallet design**. ADR 010 remains **normative for production Credits** until an Accepted cutover ADR/PR says otherwise.

### Standing rules (locked)

1. Funding Providers never define `WalletIdentity`.
2. Credit Conversion Trigger = **Confirmed Observation** only (not RPC, not Funding Provider, not address allocation alone).
3. Observation Identity = `Chain + TxHash + LogIndex` (or chain-equivalent).
4. Recovery = Seed + Index Registry; indices never reused; allocation idempotent.
5. Credits mutations only via `CreditService`; adapters never write the ledger.
6. Blockchain boundary ends at credit conversion; product ops (including auto renew) use Credits only.

### Failure Domains

Operational resilience for the Wallet + Billing system (effects are delays / unavailability — **not** silent ledger corruption):

| Failure | Effect |
|---------|--------|
| RPC / observation adapter unavailable | Observation delayed (Pending Confirmation stalls); no false Credited |
| Indexer / explorer unavailable | Same; fallbacks are ops choice, not alternate Credits SoT |
| Funding Provider unavailable | Funding UX unavailable; existing WalletAddresses and Credits unaffected |
| Wallet DB unavailable | Address allocation / Index Registry blocked; no new Observations attributed |
| Platform Wallet Infrastructure (seed/signing) unavailable | New address materialization / sweep blocked; watch may continue if DB+adapters up |
| Billing / `CreditService` unavailable | Credits delayed; Confirmed Observations wait — **must not** invent a side ledger |
| Chain reorg after Credited | Incident + Billing remediation; never silent rewrite of ledger history (RFC 006) |

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

- One Accepted (when this ADR is Accepted) implementable Wallet target.
- Clear Normative Sources table for engineers and future ADRs.
- Failure Domains set operational expectations before coding.

**Negative / cost**

- Implementation work across Wallet infra, observation, funding adapters, and cutover.
- Temporary dual-path complexity during migration from ADR 010 shared wallet.

**Neutral**

- Vision/RFC documents stay frozen; changes go through research evidence or new ADRs.

## Status

**Proposed** — Architecture Review of RFC 006 passed; this ADR is the next acceptance gate before large implementation.

When **Accepted**:

- Treat Normative Sources and standing rules as binding for Wallet PRs.
- Do not open new Wallet RFCs for this cycle without freeze-process evidence.
- Prefer capability/implementation PRs and, if needed, focused ADRs (e.g. Polygon Chain Policy numbers, schema).

## Related

- [Wallet Architecture Index](../architecture/wallet-architecture-index.md)
- [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md)
- [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) · [RFC 004](../rfcs/004-platform-wallet-infrastructure.md) · [RFC 005](../rfcs/005-funding-provider-interface.md) · [RFC 006](../rfcs/006-deposit-observation-confirmation.md)
- [ADR 010](./010-polygon-usdt-prepaid-credits.md) · [ADR 012](./012-billing-extensibility-rules.md)
- [Track 1 Exit Artifact](../architecture/wallet-sandbox-artifacts/01-wallet-address-assignment.md)
- [Cross-RFC Consistency Review](../architecture/wallet-sandbox-artifacts/02-cross-rfc-consistency-review.md)
