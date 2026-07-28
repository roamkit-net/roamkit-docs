# ADR 012: Billing extensibility rules

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Post–Faza 3 billing constitution (design freeze) |

## Context

[ADR 010](./010-polygon-usdt-prepaid-credits.md) defines the **first money path**: Polygon USDT prepaid credits owned by `billing.Account`, with an append-only ledger as source of truth and `CreditService` as the sole mutator.

[ADR 011](./011-credit-vouchers-gift-codes.md) introduces the **first additional credit source** (vouchers / gift codes). More sources will follow — gift cards, referral bonuses, affiliate rewards, cashback, loyalty points, promotional campaigns.

Without a shared contract, each new source risks re-litigating ownership, ledger rules, idempotency, events, and API surface — or silently inventing a parallel money path. This ADR does **not** add product features. It is the **normative constitution** for every future credit source.

**Hierarchy (locked):**

```text
ADR 010  — money path foundation (must not be redesigned by later credit sources)
ADR 012  — extensibility rules for all credit sources (this document)
ADR 011+ — concrete credit-source ADRs that must satisfy ADR 012
```

## Decision

Adopt a **billing extensibility contract**: any new way to grant or adjust prepaid credits must extend ADR 010 invariants and obey the mandatory rules below. New sources may add models, `LedgerReferenceType` values, events, and admin/API surfaces under `/api/v1/billing/` — they **must not** change how `CreditService`, the ledger, or Account ownership work.

### Core extension rule

> **A new credit source must not change existing financial invariants from ADR 010; it may only extend them.**

Allowed extensions (examples): new models, a new `LedgerReferenceType`, new snapshot events, feature flags, admin tools, redeem/verify APIs under `/api/v1/billing/`.

Forbidden changes (examples): alternate balance mutators, mutable ledger rows, `User`-owned money FKs, provider-side balance writes, a second public money API family, dual-entry “shadow” ledgers.

### Extension vs foundation revision

> **If a proposed credit source requires changing ADR 010 financial invariants, do not write a feature ADR — open a revision of ADR 010 first.**

| Intent | Document |
|--------|----------|
| **Extension** — new models / `LedgerReferenceType` / events / flags under existing invariants | New credit-source ADR (e.g. 011, 013…) that satisfies this ADR |
| **Foundation change** — alters `CreditService`, ledger SoT, ownership, precision, or public money API family | Revise / supersede ADR 010 (STOP on feature work until Accepted) |

This keeps “proširenje” and “promjena temelja” mechanically distinct in review.

## Mandatory invariants

These are inherited from ADR 010 and are **non-negotiable** for every credit source:

| # | Invariant |
|---|-----------|
| 1 | **`CreditService` is the sole money mutator** — no other module writes `Account.balance` or inserts ledger rows for credits/debits/refunds/adjustments. |
| 2 | **Ledger is source of truth;** `Account.balance` is a derived cache. |
| 3 | **`billing.Account` is the only financial owner** — Deposit / Ledger / Subscription / Order / future credit-source records FK → Account, never → User for money. |
| 4 | **Ledger is append-only** — INSERT only; corrections are compensating entries (`REFUND`, `ADMIN_ADJUSTMENT`, or source-specific compensating rows), never UPDATE/DELETE of ledger entries. |
| 5 | **Idempotency** — mutations keyed by `idempotency_key`; same key returns the original successful result; no duplicate ledger rows. |
| 6 | **Single DB transaction** on the money path — `select_for_update` on `Account` (+ `version` bump) together with the ledger INSERT. |
| 7 | **Snapshot domain events** (`event_version = 1`+) via the in-process bus ([ADR 005](./005-domain-events.md)); happy-path handlers must not re-fetch core money fields. |
| 8 | **Public HTTP only under** `/api/v1/billing/`. |
| 9 | **Providers never touch money** — Airalo / eSIM / chain clients have no billing imports and no balance logic. |
| 10 | **Money precision** — `Decimal(max_digits=20, decimal_places=6)` on all stored money fields. |

## Requirements for new credit sources

Every new credit source ADR (and its first implementation PR) **must** define:

| Requirement | Detail |
|-------------|--------|
| **Domain model(s)** | Explicit tables for the source lifecycle (issue, redeem/grant, revoke, audit). Soft-delete / status lifecycle preferred over hard deletes when audit matters. |
| **`LedgerReferenceType`** | One (or more, only if truly distinct) registry entry; `REFERENCE_MODELS` maps to the audit/spend record used for admin deep-links. |
| **Idempotency strategy** | Documented key format (e.g. `voucher-redeem:{redemption_id}`); concurrent retries must be safe. |
| **Snapshot event(s)** | Past-tense event with core ids, amounts, `balance_after`, `ledger_entry_id`, timestamps; `event_version`. |
| **Audit fields** | Who/what caused the credit (issuer, actor, reason, external refs as applicable). |
| **Architecture tests** | CI guards: no direct `Account.balance` writes outside `CreditService`; no billing imports from Airalo/providers; ledger append-only; source-specific bypasses rejected. |
| **Feature flag** | Required when the source is a public/product surface (e.g. `VOUCHERS_ENABLED`); ops-only sources may be admin-gated instead, but must still document the gate. |
| **Lock order** | Documented lock order when the source has its own rows that must be locked before `Account` (prevent deadlocks). |

Spend paths (order / top-up / subscription) remain Account-owned and continue to debit only through `CreditService`; new credit sources do not invent parallel spend ledgers.

## Forbidden patterns

| Forbidden | Why |
|-----------|-----|
| `Account.balance += …` / direct queryset updates outside `CreditService` | Breaks SoT and concurrency |
| UPDATE or DELETE of `CreditLedgerEntry` rows | Breaks append-only audit |
| Calling providers / Airalo with billing side effects | Couples fulfillment to money |
| Billing logic inside `integrations/*` money-adjacent clients beyond returning DTOs | Providers must stay dumb |
| Duplicating a money path (second ledger, “pending balance” table as SoT, per-order payment without Account) | Undermines ADR 010 |
| Public money endpoints outside `/api/v1/billing/` | Fragmented API contract |
| Changing `CreditService` semantics to special-case one source | Prefer source services that *call* `CreditService` |
| Financial FKs to `User` for new credit-source records | Blocks Business / Team / Reseller accounts later |

## Extension checklist

Before implementing a new credit source, the feature ADR (or PR description) must answer **yes** to all:

- [ ] Does **not** change ADR 010 financial invariants; only extends them.
- [ ] Credits/debits/refunds go **only** through `CreditService`.
- [ ] Ledger remains append-only SoT; balance remains cache.
- [ ] All money FKs point at `billing.Account`.
- [ ] New `LedgerReferenceType` (+ `REFERENCE_MODELS` when resolvable) documented.
- [ ] Idempotency key strategy documented and tested (incl. concurrency).
- [ ] Snapshot domain event(s) with `event_version` documented.
- [ ] Audit / issuer / reason fields defined where ops need them.
- [ ] Architecture tests cover bypass and import boundaries.
- [ ] Public API (if any) lives under `/api/v1/billing/` and is feature-flagged when user-facing.
- [ ] Lock order documented when source rows + Account are locked together.
- [ ] Money fields use `Decimal(20,6)`.
- [ ] Non-goals state what this source will **not** become (e.g. not a second payment provider).
- [ ] Ends with an **ADR 012 compliance** table (see below).

If any item fails, **STOP** — open/amend the credit-source ADR before coding. If the change is a foundation revision, amend ADR 010 instead.

### Mandatory section — ADR 012 compliance

Every future billing / credit-source ADR **must** end with a compliance table so review is mechanical. Template:

```markdown
## ADR 012 compliance

| Rule | Status |
|------|--------|
| CreditService only | ✅ / ❌ |
| Ledger SoT | ✅ / ❌ |
| Append-only ledger | ✅ / ❌ |
| Account owner | ✅ / ❌ |
| Idempotent | ✅ / ❌ |
| Single DB transaction + lock | ✅ / ❌ |
| Snapshot event | ✅ / ❌ |
| `/api/v1/billing/` only | ✅ / ❌ |
| Feature flag / gate | ✅ / ❌ |
| Architecture tests | ✅ / ❌ |
| Does not revise ADR 010 invariants | ✅ / ❌ |
```

Any ❌ blocks **Accepted** until resolved or escalated to an ADR 010 revision.

## Implementation Definition of Done (credit sources)

In addition to the org [Definition of Done](../../standards/definition-of-done.md), each credit-source delivery is Done only when:

- [ ] Credit-source ADR is **Accepted** (and complies with this ADR).
- [ ] Architecture tests pass in CI (money-path + source-specific bypass guards).
- [ ] OpenAPI / API docs updated when a public `/api/v1/billing/` endpoint is added.
- [ ] Metrics added for grant / redeem / fail paths (at least counters; alert hooks as applicable).
- [ ] Audit trail confirmed (issuer / actor / reason / ledger reference resolvable).
- [ ] Docs updated (`ADR_INDEX`, README/env notes, runbooks as needed).

## Non-goals

- This ADR does **not** implement vouchers, referrals, cashback, loyalty, or gift cards.
- It does not redesign ADR 010 deposit / Polygon verification.
- It does not authorize production launch ([ADR 007](./007-staging-only-until-launch.md) still gates prod).
- It is not a full accounting / double-entry GL framework.

## Consequences

### Positive

- One review checklist for every future credit source.
- ADR 010 stays the foundation; ADR 011+ stay product decisions, not money-path redesigns.
- Architecture tests and PR review share a single vocabulary (“satisfies ADR 012”).
- Compliance tables make Accept/Reject nearly mechanical.

### Negative

- Slight ceremony before each new credit source (ADR + checklist) — intentional.
- ADR numbers are not chronological by dependency (010 → 012 → 011); hierarchy is documented explicitly above.

## Related

- [ADR 005](./005-domain-events.md) — in-process domain event bus (snapshot events)
- [ADR 007](./007-staging-only-until-launch.md) — staging-only until production launch
- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — Polygon USDT prepaid credits (money path foundation)
- [ADR 011](./011-credit-vouchers-gift-codes.md) — first credit-source extension (must satisfy this ADR)
- [Definition of Done](../../standards/definition-of-done.md)
- [ADR_INDEX](../../ADR_INDEX.md)
