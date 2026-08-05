# RFC 006: Deposit Observation & Confirmation

| Field | Value |
|-------|-------|
| Status | Draft |
| Date | 2026-08 |
| Authors | Product / Engineering |
| Parent | [RoamKit Wallet Platform Vision](../architecture/roamkit-wallet-platform-vision.md) |
| Depends on | [RFC 003](./003-wallet-domain-ownership-model.md) (Frozen), [RFC 004](./004-platform-wallet-infrastructure.md) (Frozen), [RFC 005](./005-funding-provider-interface.md) (Frozen) |
| Freeze context | [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md) |
| Index | [Wallet Architecture Index](../architecture/wallet-architecture-index.md) |
| Template | [TEMPLATE-wallet.md](./TEMPLATE-wallet.md) |

> This RFC is a proposal. It does **not** amend [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md). No implementation may treat this document as normative until a related ADR is Accepted.
>
> Do not amend frozen Vision / RFC 003–005 unless [freeze rules](../architecture/wallet-architecture-freeze.md) are met.

---

## Problem

RFC 003–005 define what a Wallet is, how receive addresses are issued, and how value is guided to a `WalletAddress`. They do not define **when** an inbound transfer is trustworthy enough to convert into Credits.

Without observation & confirmation rules:

- Teams conflate “tx seen on an explorer” with “safe to credit.”
- Funding Provider status or exchange history risks becoming a second SoT.
- Chain reorganizations can double-credit or credit then reverse incorrectly.
- Adapters (RPC, indexer, explorer) leak into the domain as authority.
- Multiple Transfer logs in one EVM transaction are mistaken for one deposit.

The question this RFC answers:

> **How does the Wallet know, with sufficient confidence, that a given deposit is confirmed enough that Credit Conversion may proceed?**

Not: how to call Polygon RPC. Not: how Etherscan works. Not: how MEXC `hisrec` works. Those are **observation adapters**.

---

## Goals

- Define an **Observation State Machine** (full lifecycle, not only Confirmed).
- Define **Observation Identity** (stable, idempotent key for one inbound value event).
- Define **Confirmation Policy** as a chain-agnostic abstraction (not “Polygon = N”).
- Define **Observation Window** lifecycle (observe → timeout → archive) without hardcoding durations.
- Lock **Credit Conversion Trigger** = **Confirmed Observation** only.
- Require **idempotent** handling of duplicate observations across adapters/retries.
- State reorg / rollback behavior at policy level.
- Keep Funding Providers and Credits ledger boundaries intact.

## Non-goals

- Choosing RPC vs indexer vs explorer as the production stack (research / ADR).
- Numeric confirmation depths per chain (→ Chain Policy / Wallet ADR).
- Implementing any specific vendor SDK (Alchemy, Infura, Etherscan, MEXC, …).
- Funding Provider buy/withdraw UX (RFC 005).
- Address allocation (RFC 004).
- Amount matching / mismatch UX beyond “confirmation does not bypass Billing/`CreditService` rules.”
- Sweep / gas / treasury ops.
- Amending frozen RFC 003–005.
- Terminology cleanup of Vision/Sandbox legacy labels (deferred until this RFC is frozen).
- ADR 010 cutover details.

---

## Domain

### Standing rules (inherited)

1. Deposit → `WalletAddress` → `WalletIdentity` → Account (RFC 003).
2. Funding Providers never define `WalletIdentity` and are never Credits SoT (RFC 005).
3. Credits mutations only via Billing / `CreditService`.
4. No irreversible credit conversion until domain invariants hold (RFC 003).

### Observation vs Confirmation

| Concept | Meaning |
|---------|---------|
| **Observation** | A domain record that inbound value *may* have arrived at a watched `WalletAddress`, keyed by Observation Identity. |
| **Confirmation** | Policy decision that an Observation is **sufficient** → state **Confirmed** (eligible for Credit Conversion). |
| **Credit Conversion** | Billing step started only from a **Confirmed Observation** → eventually **Credited**. |

```text
Observation adapters          Confirmation Policy           Billing
(RPC / indexer / explorer /…)  (this RFC + Chain Policy)     (CreditService)
        │                              │                           │
        ▼                              ▼                           ▼
   Observed → Pending Confirmation → Confirmed → Conversion Started → Credited
```

### Observation Identity

What uniquely identifies one deposit observation (domain model):

```text
Observation Identity  =  Chain  +  TxHash  +  LogIndex
```

| Component | Why |
|-----------|-----|
| **Chain** | Same tx hash can exist in different network contexts; Chain scopes the event. |
| **TxHash** | On-chain transaction identifier. |
| **LogIndex** | On EVM, one transaction may emit **multiple** Transfer (or similar) logs — each is a distinct value movement. |

Rules:

- Observation Identity is the **idempotency key** for Detected/Confirmed/Credited handoff.
- Two adapter signals with the same identity **must** collapse into the **same** Observation.
- TxHash alone is **not** sufficient on EVM-class chains.

Non-EVM chains may map an equivalent triple (or documented substitute) in a future Chain Adapter ADR — the RFC requires a **stable per-transfer identity**, not EVM forever.

### Observation State Machine

Happy path:

```text
Observed
    ↓
Pending Confirmation
    ↓
Confirmed
    ↓
Credit Conversion Started
    ↓
Credited
```

Alternate paths:

```text
Observed → Rejected
Observed → Expired
Pending Confirmation → Rejected
Pending Confirmation → Expired
```

| State | Meaning |
|-------|---------|
| **Observed** | At least one adapter signal created/updated an Observation for a valid Observation Identity attributed to a RoamKit `WalletAddress`. |
| **Pending Confirmation** | Confirmation Policy is evaluating sufficiency (waiting on Chain Policy inputs). |
| **Confirmed** | Policy accepts the Observation as eligible for Credit Conversion. |
| **Credit Conversion Started** | Billing/`CreditService` convert has been requested for this Observation Identity (not yet necessarily ledger-final). |
| **Credited** | Billing completed conversion; Wallet does not own ledger rows. |
| **Rejected** | Policy or attribution rejected the Observation (wrong asset/address/rules); no credit. |
| **Expired** | Observation Window ended without reaching Confirmed (or abandoned under policy); no credit. |

Mapping to RFC 003 Deposit states (compatible, not a reopen):

| RFC 006 | RFC 003 (approx.) |
|---------|-------------------|
| Observed | Detected |
| Pending Confirmation | Confirming |
| Confirmed | Confirmed |
| Credit Conversion Started | (between Confirmed and Credited) |
| Credited | Credited |
| Rejected | Failed |
| Expired | Expired |

### Confirmation Policy (chain-agnostic)

RFC 006 does **not** say “Polygon = N confirmations.”

It defines a model:

```text
Chain Policy
    ↓
Confirmation Policy
    ↓
Confirmed
```

| Layer | Responsibility |
|-------|----------------|
| **Chain Policy** | Per-chain parameters (depth, finality tag, reorg assumptions) — supplied by Chain Adapter / ADR. |
| **Confirmation Policy** | Platform rules that consume Chain Policy + Observation evidence and decide **Pending Confirmation → Confirmed** (or Rejected). |

Confirmation Policy must specify (concrete numbers → ADR / Chain Policy, not this RFC):

1. **Sufficiency** — when evidence is enough (using Chain Policy inputs).
2. **Attribution** — maps to exactly one active or watchable (retired) `WalletAddress` (Index Registry / domain).
3. **Asset + Chain** — accepted deposit pair for that address.
4. **Amount handling** — does not invent Credits rules; Billing / ADR 010 (today) / future Wallet ADR.
5. **Identity** — uses Observation Identity for all transitions.

### Observation Window

Lifecycle concern (durations are ADR/ops, not this RFC):

| Phase | Meaning |
|-------|---------|
| **Observe** | Accept new/updated signals for an Observation Identity. |
| **Pending** | Inside Confirmation Policy evaluation. |
| **Timeout** | Window elapsed → may transition to **Expired** if not Confirmed. |
| **Archive** | Terminal Observations retained for audit; no longer active for confirmation. |

Retired `WalletAddress` watch windows (RFC 004) interact with this window but are parameterized elsewhere.

### Credit Conversion Trigger

What starts Credit Conversion / `CreditGranted` (or equivalent)?

> **A Confirmed Observation.**

| May trigger convert? | |
|----------------------|--|
| Confirmed Observation | **Yes** (only path) |
| RPC / indexer / explorer signal alone | **No** |
| Funding Provider status / webhook | **No** |
| WalletIdentity / address allocation | **No** |
| Pending Confirmation / Observed | **No** |

Handoff requirements:

1. Observation is **Confirmed**, and  
2. Observation Identity has not already started/completed credit, and  
3. Billing/`CreditService` accepts the convert request.

Adapters **must not** call `CreditService` directly.

### Duplicate Observation (idempotency)

If the same underlying event arrives via:

- RPC and Indexer, or  
- RPC retry, or  
- Webhook retry, or  
- Explorer fallback  

…it **must** resolve to the **same** Observation (same Observation Identity).

| Rule | Requirement |
|------|-------------|
| Idempotent ingest | Same identity → one Observation row / aggregate |
| Idempotent confirm | Confirming twice does not create two Confirmed outcomes |
| Idempotent credit | One Credited (or Conversion Started) per Observation Identity |

### Reorganization and rollback

| Situation | Required behavior |
|-----------|-------------------|
| Signal invalidated while **Pending Confirmation** | Do not Confirmed; may Rejected or remain Pending per policy. |
| Reorg after **Confirmed** but before **Credited** | Must not complete Credited; move out of Confirmed / hold Conversion Started. |
| Reorg after **Credited** | **Must not silently rewrite ledger history.** Incident + Billing remediation; Wallet may delay *new* credits only (RFC 003). |

Chain Policy should make post-credit reorg vanishingly rare for production chains.

### Observation adapters (pluggable, non-authoritative alone)

| Adapter class | Role |
|---------------|------|
| Chain RPC | Candidate source |
| Indexer | Candidate source |
| Explorer API | Ops / fallback — not Credits SoT |
| Funding Provider webhook/status | UX/ops only — **never** confirmation authority |

Primary adapter choice is **out of scope** (research / ADR).

### Invariants

1. Confirmation authority is **Confirmation Policy** (+ Chain Policy inputs), not a vendor API.
2. Credit Conversion Trigger = **Confirmed Observation** only.
3. Observation Identity = `Chain + TxHash + LogIndex` (or chain-equivalent).
4. Duplicate signals collapse to one Observation (idempotent).
5. Funding Provider status ≠ Confirmed ≠ Credited.
6. Adapters do not mutate Credits.
7. Post-credit ledger integrity outranks Wallet availability (RFC 003).

---

## Open Questions

1. Concrete Chain Policy for Polygon USDT (depth/finality) — Wallet ADR / ops.
2. Primary vs secondary observation adapter topology — research Exit Artifact.
3. Whether dual-source agreement is required before Confirmed.
4. Observation Window / retired-address watch durations.
5. Mapping ADR 010 amount rules into Confirmed → Credit Conversion Started.
6. Exact event names (`CreditGranted` vs internal convert request) in implementation.

## Exit Criteria

This RFC is ready for Architecture Review / freeze when:

- [ ] Observation State Machine accepted.
- [ ] Observation Identity (`Chain + TxHash + LogIndex`) accepted.
- [ ] Confirmation Policy as chain-agnostic abstraction accepted.
- [ ] Observation Window lifecycle accepted (without hardcoded durations).
- [ ] Credit Conversion Trigger = Confirmed Observation accepted.
- [ ] Duplicate Observation / idempotency accepted.
- [ ] Reorg behavior before/after Credited accepted.
- [ ] Adapters listed as non-SoT accepted.
- [ ] Open questions deferred to research/ADR without blocking the rules.
- [ ] Frozen RFC 003–005 unchanged except via freeze process.

**Next after freeze:** **Wallet ADR** (implementable architecture + Chain Policy parameters + cutover from ADR 010). Optional terminology cleanup PR (Vision/Sandbox “Deposit Key” / “Deposit Detection”) only after this RFC is frozen. No further Wallet RFCs required for this cycle.

---

## Related

- [Wallet Architecture Index](../architecture/wallet-architecture-index.md)
- [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md)
- [Cross-RFC Consistency Review](../architecture/wallet-sandbox-artifacts/02-cross-rfc-consistency-review.md)
- [RFC 003 — Deposit state machine](./003-wallet-domain-ownership-model.md)
- [RFC 004 — Platform Wallet Infrastructure](./004-platform-wallet-infrastructure.md)
- [RFC 005 — Funding Provider Interface](./005-funding-provider-interface.md)
- [Wallet Conversion Boundary](../architecture/wallet-conversion-boundary.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [TEMPLATE-wallet.md](./TEMPLATE-wallet.md)
