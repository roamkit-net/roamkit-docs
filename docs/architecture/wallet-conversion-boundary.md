# Wallet Conversion Boundary

| Field | Value |
|-------|-------|
| Status | **Architecture Frozen** (2026-08) — explanatory note (non-normative) |
| Date | 2026-08 |
| Parent | [RoamKit Wallet Platform Vision](./roamkit-wallet-platform-vision.md) |
| Freeze | [Wallet Architecture Freeze](./wallet-architecture-freeze.md) |

> This note explains **why** the blockchain stops at credit conversion. It does not amend [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md).
>
> **Architecture Freeze:** further modifications require evidence from subsequent research tracks or a new ADR proposal.

## The question everyone will ask

> Why don’t we just spend directly from the wallet on-chain for eSIM, top-ups, and auto renew?

## Short answer

Because RoamKit is a **prepaid product platform**, not a general-purpose crypto wallet that pays merchants on-chain for every purchase.

```text
Funding (chain / ramps)
  ↓
Deposit + confirm
  ↓
Convert to Credits
====================
Blockchain boundary
====================
  ↓
Credits Ledger  →  Orders / Renew / Subscriptions
```

## Why this is better for RoamKit

| Concern | Spend from chain every time | Convert once, spend Credits |
|---------|----------------------------|-----------------------------|
| Gas / MATIC for renew | Required | Not required |
| User Spending Keys | Often required or custodial signing | Never stored |
| Billing during RPC outage | Blocked | Existing Credits still spendable |
| Audit of eSIM sales | Mixed chain + commerce | Append-only ledger (`CreditService`) |
| Auto renew | Fragile (keys, gas, nonce) | Ledger debit — same as today’s prepaid model |

## What Wallet is for

1. **Receive** inbound value (to a WalletAddress).  
2. **Confirm** the deposit (detection + policy).  
3. **Convert** to Credits.  

Then the Wallet’s job for that deposit is done.

## What Wallet is not for (product v1)

- Signing subscription renewals on-chain  
- Paying Airalo (or any supplier) from the user’s deposit address per order  
- Holding User Spending Keys  

## Platform Deposit Keys (separate topic)

If RoamKit issues deposit addresses, the **platform** may hold **Platform Deposit Keys** (or use an Address Provider) to generate addresses and sweep to treasury. That is infrastructure—not “spending as the user.” See Vision § Key types and future **RFC 004 — Platform Wallet Key Management**.

## Principle

> **The blockchain boundary ends at credit conversion. All product operations execute exclusively against the Credits ledger.**

## Related

- [RoamKit Wallet Platform Vision](./roamkit-wallet-platform-vision.md)
- [RFC 003 — Wallet Domain & Ownership Model](../rfcs/003-wallet-domain-ownership-model.md)
- [ADR 010 — Polygon USDT prepaid credits](../adr/010-polygon-usdt-prepaid-credits.md)
