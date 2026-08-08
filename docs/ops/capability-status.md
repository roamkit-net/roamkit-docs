# Capability status (Billing Maturity Matrix)

Non-normative ops view of capability lifecycle. Architecture stays in ADRs.

| Capability | Design | Implemented | Production | Observed | Notes |
|------------|:------:|:-----------:|:----------:|:--------:|-------|
| Ledger / CreditService (ADR 010) | ✅ | ✅ | ✅ | ✅ | Public Billing DoD PASS — Criterion #3 |
| Deposits / Polygon verify | ✅ | ✅ | ✅ | ⏳ | On-chain Polygon SoT + ledger; customer credit never via exchange APIs. UX polish done — see [Deposit UX observation](./deposit-ux-observation.md) |
| Deposit UX (PR0–PR5) | ✅ | ✅ | ✅ | ⏳ | Telemetry, network warning, explorer, mismatch retry, CEX panel, pending resume. Promoted with deposit billing; **7-day observation** — [deposit-ux-observation.md](./deposit-ux-observation.md) |
| Billing HTTP `/api/v1/billing/` | ✅ | ✅ | ✅ | ✅ | Criterion #3 public DoD |
| WalletConnect deposit UX | ✅ | ✅ | ⏳ | ⏳ | Flag OFF at first cutover |
| Subscriptions service | ✅ | ✅ | ❌ | ❌ | `SUBSCRIPTIONS_ENABLED=false` |
| eSIM Auto Top-up v1 | ✅ | ✅ | ✅ | ❌ | Design lock: [esim-auto-topup-v1-design-lock.md](../design/esim-auto-topup-v1-design-lock.md); ≠ Subscription; does not amend ADR 010. Delivered: docs [#141](https://github.com/roamkit-net/roamkit-docs/pull/141) (`ffacbc8`), api [#76](https://github.com/roamkit-net/roamkit-api/pull/76)/[#77](https://github.com/roamkit-net/roamkit-api/pull/77)/[#78](https://github.com/roamkit-net/roamkit-api/pull/78) (`5b235c5`/`ef2439f`/`f5ac588`), web [#144](https://github.com/roamkit-net/roamkit-web/pull/144) (`433d507`). Prod promote: api [#82](https://github.com/roamkit-net/roamkit-api/pull/82) (`41b6200`), web [#147](https://github.com/roamkit-net/roamkit-web/pull/147) (`94866c5`); `AUTO_TOPUP_ENABLED=true` / `ROLLOUT_MODE=all`. Observed ❌ until metrics |
| eSIM Auto Top-up v2 | ✅ | ✅ | ✅ | ❌ | Design lock: [esim-auto-topup-v2-design-lock.md](../design/esim-auto-topup-v2-design-lock.md); OR triggers (`expiry_enabled` + `usage_mode`); supersedes v1 trigger shape only; does not amend ADR 010. Delivered: docs [#143](https://github.com/roamkit-net/roamkit-docs/pull/143) (`cb05af3`), api [#79](https://github.com/roamkit-net/roamkit-api/pull/79)/[#80](https://github.com/roamkit-net/roamkit-api/pull/80)/[#81](https://github.com/roamkit-net/roamkit-api/pull/81) (`508ebf4`/`7138ad2`/`2c4f74c`), web [#145](https://github.com/roamkit-net/roamkit-web/pull/145)/[#146](https://github.com/roamkit-net/roamkit-web/pull/146) (`90b5a71`/`5086a3f`). Prod promote: api [#82](https://github.com/roamkit-net/roamkit-api/pull/82) (`41b6200`), web [#147](https://github.com/roamkit-net/roamkit-web/pull/147) (`94866c5`). Observed ❌ until metrics |
| eSIM Auto Top-up v3 | ✅ | ✅ | ✅ | ❌ | Production promote PASS. Design lock: [esim-auto-topup-v3-design-lock.md](../design/esim-auto-topup-v3-design-lock.md); optional `active_until` + `schedule_ended`. Delivered: docs [#146](https://github.com/roamkit-net/roamkit-docs/pull/146), api [#83](https://github.com/roamkit-net/roamkit-api/pull/83)/[#84](https://github.com/roamkit-net/roamkit-api/pull/84)/[#85](https://github.com/roamkit-net/roamkit-api/pull/85) → promote [#86](https://github.com/roamkit-net/roamkit-api/pull/86) (`ac3d445`), web [#148](https://github.com/roamkit-net/roamkit-web/pull/148)/[#149](https://github.com/roamkit-net/roamkit-web/pull/149) → promote [#150](https://github.com/roamkit-net/roamkit-web/pull/150) (`85453ee`). Observed still ❌ until production observation window. |
| Credit vouchers (ADR 011) | ✅ | ❌ | ❌ | ❌ | Blocked until **after Phase 7**: [voucher-pr1-kickoff.md](./voucher-pr1-kickoff.md); [Airalo freeze](./airalo-go-live-readiness/README.md) |
| Billing extensibility (ADR 012) | ✅ | ✅ (docs) | n/a | n/a | Constitution, not a runtime feature |
| Production platform (ADR 013 PR1) | ✅ | ✅ | ✅ | ✅ | Gate D GO + Phase 4 Complete — [phase-4-complete.md](./releases/1.0.0/evidence/phase-4-complete.md) |
| `GET /version` | ✅ | ✅ | ✅ | ✅ | Verified on prod (Criteria #4 / #2) |
| Sentry / uptime | ✅ | ✅ | ✅ | ✅ | Criterion #2 PASS — [observability-criterion-2.md](./releases/1.0.0/evidence/observability-criterion-2.md); Kuma 13–17 |
| OpenAPI / schema (C10) | ✅ | ✅ | ⏳ | n/a | Staging schema/docs; prod after freeze policy |
| Billing dashboard | ✅ | ❌ | ❌ | ❌ | [billing-dashboard.md](./billing-dashboard.md) |

**Non-goals (do not start without a new capability / ADR):** multi-chain deposits; exchange/`hisrec`-style payment rails as customer credit source; ADR amount-policy changes. See [Deposit UX observation](./deposit-ux-observation.md).

**Column definitions**

- **Design** — ADR Accepted (or explicit N/A design note).
- **Implemented** — merged to `develop`, usable on staging (or docs-only constitution).
- **Production** — running on `/opt/stacks/roamkit-production/` with intended flags (or customer traffic).
- **Observed** — confirmed via metrics / [billing dashboard](./billing-dashboard.md) in production (not only flag ON).

Update at each Launch Gate milestone, Airalo go-live phase exit, when vouchers ship, and when Deposit UX observation closes.

## Related

- [Deposit UX observation](./deposit-ux-observation.md)
- [Operations Handbook](./README.md)
- [Launch Gates](./launch-gates.md)
- [Airalo Go-Live Readiness](./airalo-go-live-readiness/README.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
- [eSIM Auto Top-up v1 Design Lock](../design/esim-auto-topup-v1-design-lock.md)
- [eSIM Auto Top-up v2 Design Lock](../design/esim-auto-topup-v2-design-lock.md)
- [eSIM Auto Top-up v3 Design Lock](../design/esim-auto-topup-v3-design-lock.md)
- [ADR 013](../adr/013-production-launch.md)
- [Production readiness review](./production-readiness-review.md)
- [Go-live checklist](./production-go-live-checklist.md)
