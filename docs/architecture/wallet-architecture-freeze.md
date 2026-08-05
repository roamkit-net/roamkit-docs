# Wallet Architecture Freeze

| Field | Value |
|-------|-------|
| Status | Active |
| Date | 2026-08 |
| Scope | Wallet foundation docs |

## Frozen set

```text
Wallet Vision                 🔒
Conversion Boundary           🔒
RFC 003 — Domain & Ownership  🔒
RFC 004 — Platform Wallet Infrastructure  🔒
RFC 005 — Funding Provider Interface  🔒
RFC 006 — Deposit Observation & Confirmation  🔒
Sandbox framework             🔒  (process)
```

| Document | Status |
|----------|--------|
| [Vision](./roamkit-wallet-platform-vision.md) | Architecture Frozen |
| [Conversion Boundary](./wallet-conversion-boundary.md) | Architecture Frozen |
| [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) | Architecture Review Passed / Frozen |
| [RFC 004](../rfcs/004-platform-wallet-infrastructure.md) | Architecture Review Passed / Frozen |
| [RFC 005](../rfcs/005-funding-provider-interface.md) | Architecture Review Passed / Frozen |
| [RFC 006](../rfcs/006-deposit-observation-confirmation.md) | Architecture Review Passed / Frozen |
| [Sandbox framework](./wallet-sandbox.md) | Frozen (framework) |

## Rule

> **Further modifications require evidence from subsequent research tracks or a new ADR proposal.**

Architecture Freeze does **not** mean “never change.” It means **no change without a reason** — Exit Artifact, research decision, or ADR proposal. Prefer amending via that path over “one more small RFC tweak.”

## What is not frozen

- [ADR 017 — RoamKit Wallet Platform](../adr/017-roamkit-wallet-platform.md) (**Accepted**)
- Technical appendices (e.g. [Funding Provider Interface Contract](./funding-provider-interface-contract.md))
- [Wallet Architecture Index](./wallet-architecture-index.md) (navigation — update freely)
- Sandbox **Exit Artifacts** and Decision Register rows
- Future ADRs only when implementation evidence requires them
- Implementation in `roamkit-api` / `roamkit-web`
- Optional terminology cleanup (legacy “Platform Deposit Key” / “Deposit Detection”) — separate small PR when convenient

## Review posture after freeze

Do **not** keep reviewing frozen RFCs for incremental rules. Review:

- implementation PRs: **Is this consistent with ADR 017?**
- research evidence only if a real gap appears,
- new ADRs when deviation is required.

**RFC cycle for this Wallet foundation is complete. Architecture phase closed.** Capabilities next.

## Related

- [Wallet Architecture Index](./wallet-architecture-index.md)
- [ADR 017 — RoamKit Wallet Platform](../adr/017-roamkit-wallet-platform.md)
- [Track 1 Exit Artifact](./wallet-sandbox-artifacts/01-wallet-address-assignment.md)
- [Cross-RFC Consistency Review](./wallet-sandbox-artifacts/02-cross-rfc-consistency-review.md)
- [Funding Provider Interface Contract](./funding-provider-interface-contract.md)
