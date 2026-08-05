# Funding Provider — Interface Contract Appendix

| Field | Value |
|-------|-------|
| Status | Draft appendix (technical, non-RFC) |
| Date | 2026-08 |
| Parent | [RFC 005 — Funding Provider Interface](../rfcs/005-funding-provider-interface.md) (Frozen) |
| Freeze | Does **not** amend RFC 005; lives beside it |

Logical contract every Funding Provider adapter implements. No language, no JSON, no vendor SDK.

```text
FundingProvider

  deposit()    — guide or move Asset to the given RoamKit WalletAddress (+ Chain)
  status()     — provider-side progress for UX / ops (never Credits SoT)
  metadata()   — provider id, Asset/Chain labels, limits, errors
```

Optional capabilities (RFC 005 matrix) may extend the contract later without changing these three required operations’ meaning.

## Related

- [RFC 005 Capability Matrix](../rfcs/005-funding-provider-interface.md#provider-capability-matrix)
- [Wallet Architecture Freeze](./wallet-architecture-freeze.md)
