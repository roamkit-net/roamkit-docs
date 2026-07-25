# Voucher PR1 kickoff

**Blocked** until: Gate D exit · [Production retrospective](./production-retrospective.md) filed ·
production stable enough that feature work does not collide with a freeze.

Architecture: [ADR 011](../adr/011-credit-vouchers-gift-codes.md) (must satisfy [ADR 012](../adr/012-billing-extensibility-rules.md)).

## PR1 scope (api)

- Models + redeem path + API + tests + arch tests
- `LedgerReferenceType.VOUCHER` → `VoucherRedemption`
- No admin campaigns UI (PR2) · no web redeem UI (PR3)

## Pre-start checklist

- [ ] Gate D Hypercare closed
- [ ] Retrospective action items reviewed
- [ ] No active Production Freeze
- [ ] Maturity Matrix: vouchers Design ✅, Implemented still ❌
- [ ] Branch from `develop` (not from a merged cutover branch)

## Related

- [Launch Gates](./launch-gates.md)
- [Capability / Maturity Matrix](./capability-status.md)
- roamkit.plan.md post–Faza 3 voucher roadmap
