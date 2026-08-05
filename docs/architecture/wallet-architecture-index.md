# Wallet Architecture Index

| Field | Value |
|-------|-------|
| Status | Active (index — not an RFC) |
| Date | 2026-08 |
| Purpose | One-page map of Wallet architecture docs |
| Consistency | [Cross-RFC Consistency Review](./wallet-sandbox-artifacts/02-cross-rfc-consistency-review.md) |

This is **not** a Vision and **not** an RFC. It only points at the chain of documents and their one-line jobs.

---

## Document chain

```text
Vision
    ↓
Conversion Boundary
    ↓
RFC 003 (Domain)           🔒
    ↓
RFC 004 (Infrastructure)   🔒
    ↓
RFC 005 (Funding Interface) 🔒
    ↓
RFC 006 (Observation)      🔒
    ↓
ADR 017 (Wallet Platform)  **Accepted**
```

| Document | Status | One-line job |
|----------|--------|--------------|
| [Vision](./roamkit-wallet-platform-vision.md) | Frozen | Long-term Wallet + Credits direction (non-normative) |
| [Conversion Boundary](./wallet-conversion-boundary.md) | Frozen | Why chain stops at Credits; product ops on ledger only |
| [RFC 003 — Domain & Ownership](../rfcs/003-wallet-domain-ownership-model.md) | Frozen | What Wallet is: Account → WalletIdentity → WalletAddress → Deposit |
| [RFC 004 — Platform Wallet Infrastructure](../rfcs/004-platform-wallet-infrastructure.md) | Frozen | How addresses are allocated/recovered (HD + Index Registry) |
| [RFC 005 — Funding Provider Interface](../rfcs/005-funding-provider-interface.md) | Frozen | How value is guided to a RoamKit WalletAddress |
| [RFC 006 — Deposit Observation & Confirmation](../rfcs/006-deposit-observation-confirmation.md) | **Frozen** | When a deposit is Confirmed enough for Credit Conversion |
| [ADR 017 — RoamKit Wallet Platform](../adr/017-roamkit-wallet-platform.md) | **Accepted — Architecture Complete** | Constitution for capabilities; cutover from ADR 010 still explicit follow-on |
| [Architecture Freeze](./wallet-architecture-freeze.md) | Active | Change control for frozen set |
| [Sandbox framework](./wallet-sandbox.md) | Frozen (process) | Research tracks → Exit Artifacts |
| [Funding Provider Interface Contract](./funding-provider-interface-contract.md) | Appendix | Logical `deposit` / `status` / `metadata` |
| [Track 1 Exit Artifact](./wallet-sandbox-artifacts/01-wallet-address-assignment.md) | Closed | Evidence for RFC 004 |
| [Cross-RFC Consistency Review](./wallet-sandbox-artifacts/02-cross-rfc-consistency-review.md) | Closed | Vocabulary / authority / dependency check |
| [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md) | Accepted | Production Credits / deposit money path today |

---

## Standing rules (quick)

1. Funding Providers never define `WalletIdentity`.
2. Funding Provider / observation adapters never Credits SoT.
3. Recovery = Seed + Index Registry.
4. Blockchain boundary ends at credit conversion.

---

## Next

- Implement **small capabilities** under milestone Wallet Platform v1 (Address Allocation first).  
- Review each PR against **ADR 017** (not against inventing new RFCs).  
- Do not reopen RFC 003–006 / Vision without [freeze](./wallet-architecture-freeze.md) evidence.  
- Optional later: terminology cleanup; ADR 010 cutover plan.
