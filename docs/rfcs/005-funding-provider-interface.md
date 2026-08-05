# RFC 005: Funding Provider Interface

| Field | Value |
|-------|-------|
| Status | Draft |
| Date | 2026-08 |
| Authors | Product / Engineering |
| Parent | [RoamKit Wallet Platform Vision](../architecture/roamkit-wallet-platform-vision.md) |
| Depends on | [RFC 003](./003-wallet-domain-ownership-model.md) (Frozen), [RFC 004](./004-platform-wallet-infrastructure.md) (Frozen) |
| Freeze context | [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md) |
| Template | [TEMPLATE-wallet.md](./TEMPLATE-wallet.md) |

> This RFC is a proposal. It does **not** amend [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md). No implementation may treat this document as normative until a related ADR is Accepted.
>
> Do not amend frozen Vision / RFC 003 / RFC 004 unless [freeze rules](../architecture/wallet-architecture-freeze.md) are met.

---

## Problem

RFC 003–004 define **where** value is received (`WalletAddress` owned by RoamKit) and **how** addresses are allocated. They do not define **how value is guided toward that address** from the outside world.

Without a Funding Provider interface:

- Product UX risks embedding a single exchange (MEXC, Binance, …) into the Wallet domain.
- Teams confuse “user bought USDT on an exchange” with “Credits were granted.”
- Card on-ramps and CEX withdrawals look like separate products instead of interchangeable adapters.

The question this RFC answers:

> **How does value arrive at a RoamKit `WalletIdentity` (via its active `WalletAddress`)?**

Not: how RPC works. Not: how one vendor’s API works in isolation.

---

## Goals

- Define **Funding Provider** as a pluggable **adapter**, not a domain owner.
- Separate **FundingSource** (domain kind of funding) from **FundingProvider** (concrete integration) — per RFC 003.
- Specify the **destination contract**: providers deliver value to a RoamKit-owned `WalletAddress`.
- State standing invariants inherited from the freeze (never define `WalletIdentity`; never Credits SoT).
- Describe provider **capabilities** at interface level (guide / buy / withdraw-to-address / status), without vendor lock-in.
- Keep Deposit Detection and Credits conversion out of this interface.

## Non-goals

- Deposit Detection (RPC / indexer / explorer) — RFC 006 / research.
- Choosing a single vendor (MEXC vs Binance vs MoonPay) as architecture.
- Card processor contracts, KYC product design, or fee schedules (ops / vendor research).
- Amending RFC 003 / 004 / Vision.
- Credits grant from provider webhooks or `hisrec`.
- Withdrawals from RoamKit to users; swap; staking; multi-chain beyond “Polygon first.”
- ADR 010 cutover.

---

## Domain

### Standing rules (inherited, not reopened)

1. **Funding Providers never define `WalletIdentity`.**
2. Destination of funded value is a RoamKit **`WalletAddress`** (RFC 004 allocation).
3. **Credits / ledger** remain Billing; provider success ≠ credit granted.
4. Blockchain / ramp activity ends at deposit → convert (Vision Funding Boundary).

### FundingSource vs FundingProvider

| | FundingSource | FundingProvider |
|---|---------------|-----------------|
| Layer | Domain (RFC 003) | Integration (this RFC) |
| Question | *What kind of funding is this?* | *Which adapter fulfilled it?* |
| Examples | On-chain transfer, exchange withdrawal, card purchase | MEXC, Binance, OKX, MoonPay, Transak, “external wallet send” (null provider) |

Same `FundingSource` may be fulfilled by many providers over time without changing WalletIdentity or Index Registry.

### Arrival path (conceptual)

```text
User
  │
  ▼
Funding Provider (adapter)     ← optional UX / rails
  │
  │  guides or moves Asset
  ▼
RoamKit WalletAddress (active) ← RFC 004
  │
  ▼
Deposit Detection              ← RFC 006 (out of scope here)
  │
  ▼
Convert → Credits              ← Billing / CreditService
```

External wallet send is the same arrival path with **no** commercial Funding Provider (user is the sender).

### Interface responsibilities

A Funding Provider adapter **may**:

| Capability | Meaning |
|------------|---------|
| **Present destination** | Show or deep-link the user’s active RoamKit `WalletAddress` + Chain + Asset expectations |
| **Guide buy** | Help the user acquire Asset (e.g. card → USDT) on the provider |
| **Guide withdraw / send** | Instruct or initiate send of Asset **to** that `WalletAddress` on the supported Chain |
| **Report provider-side status** | Optional: “buy completed”, “withdrawal submitted” — **ops/UX only**, not Credits SoT |
| **Fail clearly** | Unsupported network, limits, KYC blocked — without inventing a WalletIdentity |

A Funding Provider adapter **must not**:

- Create or own `WalletIdentity` / Index Registry rows.
- Allocate RoamKit derivation indices.
- Call `CreditService` or mutate the ledger.
- Claim deposit finality for Credits (that is Detection + Billing policy).
- Require RoamKit to treat provider balance or history as source of truth.

### Destination contract

Before any provider flow that moves funds:

1. Ensure `WalletIdentity` exists (domain).
2. Ensure an **active** `WalletAddress` for the target Chain (RFC 004 allocation — idempotent).
3. Pass **address + chain + asset** into the provider UX or API as the sole RoamKit destination.

Provider-specific deposit addresses / memos are **not** RoamKit `WalletAddress` records and must not replace them.

### Provider identity in domain

If a Deposit or funding attempt records a provider:

- Store an opaque **provider id** (or null for external send) alongside `FundingSource`.
- Do not embed vendor SDK types in the Wallet domain model.

### Minimal capability matrix (for evaluating adapters)

Use this when researching vendors (Sandbox track / Exit Artifact) — not as a product commitment to any row:

| Capability | Needed for “buy then withdraw to RoamKit”? | Credits SoT? |
|------------|--------------------------------------------|--------------|
| Fiat → Asset (card/bank) | Often yes | No |
| Withdraw Asset to external address | **Required** for CEX-style path | No |
| Correct Chain support (e.g. Polygon USDT) | **Required** for that Asset/Chain pair | No |
| Webhook / order status | Optional UX | No |
| Unique per-user deposit address **on the exchange** | Not required for RoamKit core | No |

---

## Open Questions

1. First production adapter shortlist (research Exit Artifact — not this RFC’s job to pick).
2. Whether RoamKit hosts in-app buy (embedded on-ramp) vs deep-link to provider.
3. How much provider status to surface in UI before on-chain confirm.
4. Mapping of provider “networks” labels to RoamKit `Chain` + `Asset` (ops catalog).
5. Idempotency keys for “start funding session” UX (product), distinct from address allocation idempotency (RFC 004).

## Exit Criteria

This RFC is ready to close / promote toward an ADR when:

- [ ] FundingSource ≠ FundingProvider accepted as the integration boundary.
- [ ] Destination contract (active RoamKit `WalletAddress`) accepted.
- [ ] Standing rules (never WalletIdentity; never Credits SoT) accepted.
- [ ] Capability allow/deny lists accepted at interface level.
- [ ] Open questions deferred to vendor research / ADR without blocking the interface shape.
- [ ] No production vendor hard-coded as architecture.
- [ ] Frozen RFC 003 / 004 unchanged except via freeze process.

**Next:** vendor/UX research Exit Artifact(s) against this interface; RFC 006 Deposit Detection when detection authority is the open question — not before the destination contract is clear.

---

## Related

- [Wallet Architecture Freeze](../architecture/wallet-architecture-freeze.md)
- [RFC 003 — Domain & Ownership](./003-wallet-domain-ownership-model.md)
- [RFC 004 — Platform Wallet Infrastructure](./004-platform-wallet-infrastructure.md)
- [Track 1 Exit Artifact](../architecture/wallet-sandbox-artifacts/01-wallet-address-assignment.md)
- [Vision — Funding Provider](../architecture/roamkit-wallet-platform-vision.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [TEMPLATE-wallet.md](./TEMPLATE-wallet.md)
