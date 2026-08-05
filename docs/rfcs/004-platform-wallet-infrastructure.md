# RFC 004: Platform Wallet Infrastructure

| Field | Value |
|-------|-------|
| Status | Draft |
| Date | 2026-08 |
| Authors | Product / Engineering |
| Parent | [RoamKit Wallet Platform Vision](../architecture/roamkit-wallet-platform-vision.md) |
| Also | [Wallet Conversion Boundary](../architecture/wallet-conversion-boundary.md), [RFC 003](./003-wallet-domain-ownership-model.md) |
| Evidence | [Track 1 Exit Artifact — WalletAddress Assignment](../architecture/wallet-sandbox-artifacts/01-wallet-address-assignment.md) |
| Template | [TEMPLATE-wallet.md](./TEMPLATE-wallet.md) |

> This RFC is a proposal. It does **not** amend [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md). No implementation may treat this document as normative until a related ADR is Accepted.

This RFC is driven **only** by Track 1 Architectural Consequences. It does not reopen Vision debates or introduce Deposit Detection / Funding Provider interface design (RFC 005–006).

---

## Problem

RFC 003 defines `Account → WalletIdentity → WalletAddress` but does not specify **how** RoamKit issues and recovers receive addresses, or **who owns which piece of platform state**.

Without that:

- Address assignment risks leaking into a Funding Provider (exchange-as-core).
- Recovery is mis-specified as “restore the seed” without durable index allocation.
- Key custody is framed as “HSM” when research shows HSM ≠ BIP32 HD wallet.

Track 1 closed with: **in-house HD Preferred**, Index Registry as platform state, Funding Providers never define `WalletIdentity`. This RFC turns those consequences into infrastructure requirements.

---

## Goals

- Define **Platform Wallet Infrastructure** (not “Platform Deposit Key” / not “HSM implementation”).
- Specify v1 address assignment: **in-house HD** with one **active** `WalletAddress` per Account + Chain (Polygon first).
- Make **Index Registry** a domain/platform requirement for recovery.
- Clarify **Platform State Ownership** (seed vs registry vs domain vs billing).
- State invariants: Funding Providers never define `WalletIdentity`; User Spending Keys never stored.
- Record deferred / future paths (custody, MPC/TEE) without selecting them for v1.
- Stay implementation-light: no schema migration, no production code in this RFC.

## Non-goals

- Deposit Detection (RPC / indexer / explorer) — RFC 006 / next research track.
- Funding Provider interface (MEXC, Binance, MoonPay, …) — RFC 005.
- Withdrawals to users, swap, staking, multi-chain beyond Polygon-first.
- User Spending Keys; on-chain auto renew; Credits mutations outside `CreditService`.
- Selecting Fireblocks / Turnkey / DFNS as day-one runtime.
- Amending ADR 010 or cutting over production from shared platform wallet.

---

## Domain

### Platform Wallet Infrastructure (definition)

**Platform Wallet Infrastructure** is the set of capabilities RoamKit uses to:

1. Allocate and materialize **receive** `WalletAddress` records for an Account’s `WalletIdentity`.
2. Hold cryptographic material needed to **derive** addresses and (later) **sweep** inbound funds to treasury.
3. Recover addresses from backup using **Seed + Index Registry**.
4. Evolve custody over time (wrapped seed → enclave/MPC provider) without changing the domain model.

It is **not**:

- User Spending Key storage.
- Credits / ledger logic.
- A Funding Provider.
- Synonymous with “deploy an HSM.”

### Decisions locked by Track 1

| Decision | Status |
|----------|--------|
| In-house HD as v1 assignment mechanism | **Accepted (for this RFC)** |
| One active address per Account + Chain | **Accepted** |
| Chain (Polygon) is metadata on `WalletAddress`; same EVM address bytes as ETH format (coin type 60) | **Accepted** |
| Index Registry is platform state | **Accepted** |
| Funding Providers never define `WalletIdentity` | **Accepted** (standing rule) |
| Fireblocks-class custody | **Deferred** (current product scope) |
| Turnkey / DFNS (TEE/MPC) | **Future migration candidates** |
| Funding Provider addresses as core assignment | **Not core architecture** |

### Address assignment (v1)

```text
Account
  └── WalletIdentity                 (domain; provider-agnostic)
        └── WalletAddress[]          (Polygon first)
              active | retired
```

| Step | Behavior |
|------|----------|
| Materialize | Lazy on first funding intent (“Add funds”): allocate next `derivation_index` → derive address → persist in Index Registry + domain `WalletAddress` |
| Path (suggested) | BIP44 EVM: `m/44'/60'/0'/0/{derivation_index}` |
| Active set | Exactly one **active** address per Account + Chain in v1 |
| Rotate | New index → new active; previous → **retired**, still watchable for late deposits |
| Never (v1) | New address per payment |

`WalletIdentity` is created as a domain object (no seed access required). Address materialization requires Platform Wallet Infrastructure.

### Recovery model

```text
Seed  +  Index Registry  →  WalletAddress
```

| Component | Required for recovery? |
|-----------|------------------------|
| Master seed (or equivalent root) | Yes |
| Index Registry (Account / WalletIdentity ↔ derivation_index ↔ address + lifecycle) | **Yes** |
| Funding Provider records | No |

**Architectural requirement:** index allocation is platform state. “I have the seed” alone is **not** a complete recovery story.

### Platform State Ownership

| State | Owner | Notes |
|-------|-------|-------|
| Seed (master / root material) | **Platform Wallet Infrastructure** | Encrypted at rest; access audited; never User Spending Keys |
| Index Registry | **Platform Database** (Wallet infra owns the schema concern) | Durable map; backup with DB |
| WalletIdentity | **Domain Model** (RFC 003) | Provider-agnostic |
| WalletAddress | **Domain Model** (RFC 003) | Chain is metadata; lifecycle active/retired |
| Credits balance cache | **Billing** | Via `CreditService` only |
| Ledger | **Billing** | Append-only SoT for money after convert |
| Funding Provider session / KYC / buy UX | **Funding Provider adapter** (RFC 005) | Never owns WalletIdentity |

Boundary rule:

> After convert-to-Credits, product operations (including auto renew) use **Credits only**. Platform Wallet Infrastructure does not participate in subscription spend.

### Invariants

1. RoamKit owns `WalletAddress` attribution; Funding Providers never define `WalletIdentity`.
2. Recovery requires Seed **and** Index Registry consistency.
3. No User Spending Keys are generated or stored for billing/renew.
4. Platform Wallet Infrastructure may hold material only for receive/derive/sweep — not for acting “as the user.”
5. Credits ledger remains independent; conversion calls into Billing/`CreditService` (details outside this RFC).
6. Evolution of custody (wrap → TEE/MPC) must preserve published addresses or define an explicit cutover + watch window — not silent renumbering.

### Evolution (not “HSM implementation”)

| Phase | Intent |
|-------|--------|
| v1 | In-house HD; seed wrapped (e.g. KMS); derive in controlled runtime |
| Hardening | Stronger wrap, dual control, recovery drills — still HD + Index Registry |
| Optional later | Migrate root/signing to Turnkey / DFNS / Fireblocks-class **if** product scope warrants — prefer HD-import paths that preserve addresses |

**Do not** specify RFC exit as “HSM done.” AWS CloudHSM does not provide BIP32 HD trees; HSM/KMS may wrap or hold individual keys, but that is an implementation choice under Platform Wallet Infrastructure evolution.

### Deferred and out of scope (consequences)

| Item | Treatment |
|------|-----------|
| Fireblocks-class custody | Deferred due to current product scope (Credits-only spend path) |
| MPC/TEE day-one | Future migration candidate |
| Deposit Detection | Out of scope → RFC 006 / research track |
| Funding Provider API | Out of scope → RFC 005 |
| Withdraw / multi-chain / user keys | Out of scope |

---

## Open Questions

1. Exact Index Registry schema fields and uniqueness constraints (→ ADR when implementing).
2. Who may authorize address **rotation** (ops vs user vs automated policy).
3. Watch window length for **retired** addresses.
4. Sweep destination (treasury) and gas-funding policy for ERC-20 receive addresses.
5. Secret ceremony / dual-control details for seed generation and restore drills.
6. Whether v1 materializes `WalletIdentity` at Account creation or only at first funding intent.
7. Cutover plan from ADR 010 shared `POLYGON_PLATFORM_WALLET` to per-Account addresses (separate ADR).

## Exit Criteria

This RFC is ready to close / promote toward an ADR when:

- [ ] Platform Wallet Infrastructure definition accepted (not “HSM RFC”).
- [ ] Platform State Ownership table accepted.
- [ ] HD + Index Registry recovery model accepted.
- [ ] One active address per Account + Chain accepted for v1.
- [ ] Standing rule accepted: Funding Providers never define `WalletIdentity`.
- [ ] Deferred custody/MPC paths recorded without blocking v1.
- [ ] Open questions answered or explicitly deferred to ADR / RFC 005–006.
- [ ] No production code required to accept this RFC as Draft→Ready.

**Next after acceptance:** implement via a dedicated Wallet ADR (cutover from ADR 010); then RFC 005 (Funding Provider Interface) and Deposit Detection research / RFC 006 — in that product order, not before this ownership model is clear.

---

## Related

- [Track 1 Exit Artifact](../architecture/wallet-sandbox-artifacts/01-wallet-address-assignment.md)
- [RFC 003 — Wallet Domain & Ownership Model](./003-wallet-domain-ownership-model.md)
- [Wallet Conversion Boundary](../architecture/wallet-conversion-boundary.md)
- [RoamKit Wallet Platform Vision](../architecture/roamkit-wallet-platform-vision.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [TEMPLATE-wallet.md](./TEMPLATE-wallet.md)
