# ADR 010: Polygon USDT prepaid credits

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Faza 3 design lock |

## Context

Faza 3 needs prepaid credits so users can deposit USDT, hold a balance, and spend it on orders, top-ups, and (later) subscriptions. Early product drafts assumed multi-chain checkout (TRC-20 / ERC-20) and a `PaymentProvider` that pays per order.

That model conflicts with prepaid balance, shared accounts for future Business / Team / Reseller use, and a single auditable money path. Without locked ownership and ledger rules, billing FKs and balance mutations would sprawl across apps.

## Decision

Adopt **Polygon-only USDT prepaid credits** owned by `billing.Account`, with an append-only ledger as source of truth.

### Network and precision

- **Network:** Polygon only (`chain_id = 137`). No TRC-20, ERC-20, or BSC in v1. No `network` column on deposit/ledger models.
- **Precision:** `Decimal(max_digits=20, decimal_places=6)` on all money fields. UI may display fewer places (e.g. `15.50`).

### Ownership

```text
User  →  Account  →  Deposit | Ledger | Subscription | Order
```

- `billing.Account` is the **only** financial owner.
- `DepositRequest`, `CreditLedgerEntry`, `Subscription`, and `Order` FK → `billing.Account`, **never** → `User`.
- Auth resolves `request.user.billing_account` (e.g. “my orders” = `Order.objects.filter(account=request.user.billing_account)`).
- `wallet_address` stays on `User` (WalletConnect identity only).

### Schema (summary)

| Model | Notes |
|-------|--------|
| `Account` | UUID PK; OneToOne `user`; `balance` Decimal(20,6); `version`; timestamps |
| `DepositRequest` | UUID PK; `account` FK; amounts Decimal(20,6); method; `tx_hash` unique; `idempotency_key` unique; status; `verified_at`; `raw_rpc_response` |
| `CreditLedgerEntry` | UUID PK; append-only; `delta`; `balance_after`; `reference_type`; `reference_id`; `idempotency_key` unique; `created_at` only |
| `Subscription` | UUID PK; `account` FK; `esim` FK; price Decimal(20,6); `next_billing_date`; status |
| `Order` | Replace `user` FK with `account` FK → `billing.Account` (data migration: each order’s user → their `billing.Account`) |

DB **CHECK** constraints:

| Model | Constraint |
|-------|------------|
| `Account` | `balance >= 0` |
| `Account` | `version >= 0` |
| `DepositRequest` | `amount_requested > 0` |
| `DepositRequest` | `amount_credited IS NULL OR amount_credited > 0` |
| `CreditLedgerEntry` | `delta != 0` |
| `Subscription` | `price_per_period > 0` |

### `LedgerReferenceType` + registry

```python
class LedgerReferenceType(models.TextChoices):
    DEPOSIT = "deposit", "Deposit"
    ORDER = "order", "Order"
    TOPUP = "topup", "Top-up"
    SUBSCRIPTION = "subscription", "Subscription"
    REFUND = "refund", "Refund"
    ADMIN_ADJUSTMENT = "admin_adjustment", "Admin adjustment"

REFERENCE_MODELS = {
    LedgerReferenceType.DEPOSIT: DepositRequest,
    LedgerReferenceType.ORDER: Order,
    LedgerReferenceType.TOPUP: ...,  # top-up record or order id convention — document when implemented
    LedgerReferenceType.SUBSCRIPTION: Subscription,
    # REFUND / ADMIN_ADJUSTMENT: reference_id may point at order/deposit/ticket; registry optional
}
```

Admin uses `REFERENCE_MODELS` to deep-link related objects when resolvable.

### Money path

- **Ledger = source of truth;** `Account.balance` is a **cache** derived from ledger.
- Ledger is **append-only** (INSERT only). Corrections are compensating rows, never UPDATE/DELETE of ledger entries.
- **`CreditService`** is the sole mutator of balance/ledger: `select_for_update` on `Account`, INSERT ledger row, bump `balance` + `version`, idempotent on `idempotency_key`.
- Repeated requests with the same `idempotency_key` return the original successful result and must not create new ledger entries.
- `ADMIN_ADJUSTMENT` / `REFUND` only via `CreditService`.
- Order / top-up spend: debit in the same DB transaction as order reserve; fulfillment failure → compensating `REFUND` credit + failed status + snapshot event.
- Airalo / other eSIM integrations have **zero** billing knowledge (must not import `apps.billing`).

### Blockchain

- `BlockchainProvider` protocol → `PolygonProvider` implementation.
- RPC: 10s timeout, 3 retries, exponential backoff.
- Deposit UX: Reown AppKit only (when `WALLETCONNECT_ENABLED`), EIP-681 QR, and CEX TXID verification.
- Env (representative): `POLYGON_RPC_URL`, `POLYGON_USDT_CONTRACT`, `POLYGON_PLATFORM_WALLET`, `POLYGON_CHAIN_ID=137`, `POLYGON_MIN_CONFIRMATIONS`, timeout/retry settings.

### API and flags

- Public billing HTTP surface: **only** `/api/v1/billing/` (balance, deposit-info, verify-wallet, verify-cex).
- Feature flags: `BILLING_ENABLED`, `SUBSCRIPTIONS_ENABLED`, `WALLETCONNECT_ENABLED`.

### Domain events (snapshots)

Billing domain events carry **snapshots** of core fields (ids, amounts, `balance_after`, `tx_hash`, etc.). Handlers must not re-fetch those fields for the happy path. Publish via the existing in-process bus ([ADR 005](./005-domain-events.md)); side effects (email, analytics) live in handlers only.

Every billing event includes `event_version = 1` so payloads can evolve without breaking handlers.

Example payloads:

```python
DepositVerified(
    event_version=1,
    deposit_id=...,
    account_id=...,
    amount=...,
    balance_after=...,
    tx_hash=...,
    payment_method=...,
    ledger_entry_id=...,
    verified_at=...,
)

CreditGranted(
    event_version=1,
    account_id=...,
    amount=...,
    balance_after=...,
    reference_type=...,
    reference_id=...,
    ledger_entry_id=...,
    created_at=...,
)
```

### Operational follow-ups (same decision set)

- Beat subscription renew with `select_for_update`; pause + email event if underfunded.
- `billing_reconcile_balances` (daily, alert on drift). Automatic balance correction is forbidden in production; rebuilding balances is an explicit operator action.
- `billing_rebuild_balances` (explicit rebuild from ledger `SUM`).
- Admin: deposit filter / re-verify; adjustments only via `CreditService`; resolve `reference_type` + `reference_id` through `REFERENCE_MODELS`.

## Financial invariants

1. **Ledger is source of truth;** `Account.balance` is a cache.
2. **Ledger is append-only** — INSERT only; corrections = compensating entries.
3. **Only `CreditService` mutates money** — no direct `Account.balance` writes elsewhere.
4. **Idempotent** mutations keyed by `idempotency_key` — same key returns the original result; no duplicate ledger rows.
5. **Single DB transaction** + `select_for_update` (+ `Account.version`) on all money paths.
6. **Providers never touch money** — Airalo/eSIM integrations have no billing imports or balance logic.
7. **Six decimal places** (`Decimal(20,6)`) everywhere money is stored.
8. **HTTP API only under** `/api/v1/billing/`.
9. **Account-only FKs** for Deposit, Ledger, Subscription, Order — never `User` as financial owner.

## Non-goals

- Not a full accounting engine (no double-entry chart of accounts, GL close, etc.).
- Not a crypto wallet for users.
- No private key storage.
- No custody of user funds beyond receiving USDT to the platform deposit address.
- No on-chain settlement between users.
- No exchange / swap functionality.
- Polygon is the only supported network in v1.

### Out of scope (v1)

- Multi-user Account membership tables (FKs are already Account-centric for later).
- Other chains, HD deposit addresses, mempool watchers, fiat/Stripe, Kafka.
- Auto-purge of `raw_rpc_response`.

## Consequences

### Positive

- One money path; audit via ledger; safe concurrency via row lock + version.
- Account-centric FKs allow Business / Team / Reseller / API accounts later without rewriting billing ownership.
- Snapshot events keep handlers decoupled from re-query races.
- Clear non-goals prevent scope creep into wallet/custody/exchange features.

### Negative

- Migrating existing `Order.user` → `Order.account` requires a careful data migration.
- Polygon-only means users on other chains need an off-platform bridge/CEX hop.
- Cached `balance` can drift if invariants are violated — mitigated by reconcile/rebuild jobs and architecture tests.

### Architecture tests

Architecture tests enforce that no module outside `apps.billing` imports billing models for mutation or writes `Account.balance` directly. Ledger immutability after create and the Airalo ↔ billing import boundary are covered the same way so CI rejects regressions of the money-path rules.

### Implementation note — `Order.account` migration

1. Introduce `billing.Account` (and OneToOne from `User`).
2. Backfill one `Account` per existing user (zero balance unless otherwise decided).
3. Add nullable `Order.account`, populate from `order.user.billing_account`, then make non-null and drop `Order.user` (or keep `user` only if a non-billing display path still needs it — prefer drop for a single ownership rule).
4. Update list/auth queries to filter by `account`.

## Related

- [ADR 004](./004-provider-interfaces.md) — provider protocols (billing uses `BlockchainProvider`, not Airalo)
- [ADR 005](./005-domain-events.md) — domain event bus
- [Provider abstractions](../architecture/provider-abstractions.md)
- RFC [001-self-service-esim-flow](../rfcs/001-self-service-esim-flow.md)
