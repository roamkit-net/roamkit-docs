# RFC 006: Deposit Observation & Confirmation

| Field | Value |
|-------|-------|
| Status | Draft |
| Date | 2026-08 |
| Authors | Product / Engineering |
| Parent | [RoamKit Wallet Platform Vision](../architecture/roamkit-wallet-platform-vision.md) |
| Depends on | [RFC 003](./003-wallet-domain-ownership-model.md) (Frozen), [RFC 004](./004-platform-wallet-infrastructure.md) (Frozen), [RFC 005](./005-funding-provider-interface.md) (Frozen) |
| Freeze context | [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md) |
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

The question this RFC answers:

> **How does the Wallet know, with sufficient confidence, that a given deposit is confirmed enough that Credit Conversion may proceed?**

Not: how to call Polygon RPC. Not: how Etherscan works. Not: how MEXC `hisrec` works. Those are **observation adapters**.

---

## Goals

- Define **observation** vs **confirmation** as distinct responsibilities.
- Align with RFC 003 Deposit state machine (`Detected` → `Confirming` → `Confirmed` → `Credited`).
- State what is **authoritative** for confirmation (rules), vs what is an **adapter** (sources).
- Define when **`CreditGranted` / convert** may be triggered.
- Require **idempotent** confirmation and credit handoff.
- State reorg / rollback behavior at policy level.
- Keep Funding Providers and Credits ledger boundaries intact.

## Non-goals

- Choosing RPC vs indexer vs explorer as the production stack (research / ADR).
- Implementing any specific vendor SDK.
- Funding Provider buy/withdraw UX (RFC 005).
- Address allocation (RFC 004).
- Amount matching / mismatch UX policy beyond “confirmation does not bypass Billing/`CreditService` rules.”
- Sweep / gas / treasury ops.
- Amending frozen RFC 003–005.
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
| **Observation** | Evidence that value *may* have arrived at a watched `WalletAddress` (candidate deposit). |
| **Confirmation** | Policy decision that evidence is **sufficient** for the Deposit to become **Confirmed** (eligible for convert). |
| **Credit Conversion** | Billing step `Confirmed` → `Credited` (outside Wallet adapters; emits / consumes platform events). |

```text
Observation adapters          Confirmation policy           Billing
(RPC / indexer / explorer /…)  (this RFC)                    (CreditService)
        │                              │                           │
        ▼                              ▼                           ▼
   DepositDetected  →  Confirming  →  Confirmed  →  CreditGranted / Credited
```

### Authoritative event (logical)

The **authoritative outcome** for Wallet→Credits handoff is:

> Deposit reaches **Confirmed** under platform confirmation policy.

Observation adapters produce **candidates** and **signals**. They do **not** grant Credits. Funding Provider status does **not** grant Credits.

### Deposit lifecycle (confirmation focus)

Inherited from RFC 003; this RFC owns the meaning of the middle states:

| State | Meaning under this RFC |
|-------|------------------------|
| **Detected** | At least one observation signal associated a transfer with a RoamKit `WalletAddress`. |
| **Confirming** | Policy is accumulating / waiting for sufficiency (e.g. depth, finality heuristic). |
| **Confirmed** | Policy accepts the deposit as eligible for Credit Conversion. |
| **Credited** | Billing has completed conversion (Wallet does not own ledger rows). |
| **Failed / Expired** | Observation abandoned or rejected under policy (no credit). |

### Confirmation policy requirements

RFC 006 requires a confirmation policy to define (concrete numbers → ADR / ops):

1. **Sufficiency** — what makes a deposit Confirmed (e.g. confirmations depth, finality tag, dual-source agreement).
2. **Attribution** — deposit must map to exactly one active or watchable (retired) `WalletAddress` in the Index Registry / domain.
3. **Asset + Chain** — observed asset/chain must match an accepted deposit pair for that address.
4. **Amount handling** — confirmation does not invent Credits rules; amount acceptance remains Billing / product policy (ADR 010 today; future Wallet ADR).
5. **Idempotency key** — stable identity for the on-chain (or equivalent) transfer so Detected/Confirmed/Credited cannot double-apply.

### Reorganization and rollback

| Situation | Required behavior |
|-----------|-------------------|
| Signal later invalidated (reorg, dropped tx) while **Confirming** | Do not Confirmed; may Failed or remain Confirming per policy. |
| Reorg after **Confirmed** but before **Credited** | Must not Credited; revert or hold Deposit out of Confirmed. |
| Reorg discovered after **Credited** | **Must not silently rewrite ledger history.** Escalate as incident; remediation via Billing/ops playbooks — Wallet outage may delay *new* credits, never corrupt past Credits (RFC 003 invariant). |

Prefer confirmation thresholds that make post-credit reorg vanishingly rare for the chosen Chain.

### CreditGranted handoff

`CreditGranted` (or equivalent convert trigger) may fire **only when**:

1. Deposit is **Confirmed**, and  
2. Idempotency key has not already produced a credit, and  
3. Billing/`CreditService` accepts the convert request.

Observation adapters **must not** call `CreditService` directly. They emit observation signals into the Wallet deposit pipeline.

### Observation adapters (pluggable, non-authoritative alone)

| Adapter class | Role |
|---------------|------|
| Chain RPC | Primary candidate source |
| Indexer | Scale / reliability candidate source |
| Explorer API | Ops / fallback signal — not Credits SoT |
| Funding Provider webhook/status | UX/ops only — **never** confirmation authority |

Selecting which adapter is primary is **out of scope** for this RFC (research track / ADR).

### Invariants

1. Confirmation authority is **platform policy**, not a vendor API.
2. Funding Provider status ≠ Confirmed ≠ Credited.
3. Confirmation and credit handoff are **idempotent** on a stable transfer identity.
4. Adapters do not mutate Credits.
5. Post-credit ledger integrity outranks Wallet availability (RFC 003).

---

## Open Questions

1. Exact confirmation depth / finality rule for Polygon USDT (ADR / ops).
2. Primary vs secondary observation adapter topology (research Exit Artifact).
3. Whether dual-source agreement is required before Confirmed.
4. Watch window for **retired** addresses (ties to RFC 004 open question).
5. Mapping of ADR 010 exact-amount rules into Confirmed→Credited under Wallet cutover.

## Exit Criteria

This RFC is ready to close / promote toward an ADR when:

- [ ] Observation vs Confirmation vs Credit Conversion accepted.
- [ ] Authoritative outcome = Deposit **Confirmed** under policy accepted.
- [ ] Reorg behavior before/after Credited accepted.
- [ ] Idempotent handoff to `CreditService` accepted.
- [ ] Adapters listed as non-SoT (including Funding Provider status) accepted.
- [ ] Open questions deferred to research/ADR without blocking the rules.
- [ ] Frozen RFC 003–005 unchanged except via freeze process.

**Next:** observation-adapter research Exit Artifact if needed; then Wallet ADR covering confirmation parameters + cutover from ADR 010 shared wallet.

---

## Related

- [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md)
- [RFC 003 — Deposit state machine](./003-wallet-domain-ownership-model.md)
- [RFC 004 — Platform Wallet Infrastructure](./004-platform-wallet-infrastructure.md)
- [RFC 005 — Funding Provider Interface](./005-funding-provider-interface.md)
- [Wallet Conversion Boundary](../architecture/wallet-conversion-boundary.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [TEMPLATE-wallet.md](./TEMPLATE-wallet.md)
