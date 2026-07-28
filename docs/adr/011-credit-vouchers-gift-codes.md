# ADR 011: Credit vouchers & gift codes

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Post–Faza 3 design lock (billing design freeze) |

## Context

After Polygon USDT prepaid credits ([ADR 010](./010-polygon-usdt-prepaid-credits.md)), product needs a second way to top up `billing.Account` balances: promotional codes, gift vouchers, partner/admin grants.

A voucher must not become a second payment provider or a parallel money path. Credits must still flow only through `CreditService`, with the ledger as source of truth and `Account` as the only financial owner.

Without an explicit model for shared campaign codes vs unique gift codes, batch issuance, revoke audit, and lock ordering, later PRs would reshape schema and `LedgerReferenceType` after Faza 3 ships.

## Decision

Adopt **credit vouchers & gift codes** as a new **credit source** inside `apps.billing`, fully subordinate to ADR 010 invariants and the billing extensibility contract ([ADR 012](./012-billing-extensibility-rules.md)).

This ADR must satisfy the ADR 012 extension checklist before implementation. It **extends** ADR 010 (new models, `LedgerReferenceType.VOUCHER`, events); it must **not** change `CreditService`, ledger append-only rules, or Account ownership.

### Relationship to ADR 010 / ADR 012

- Voucher is **not** a payment provider and **not** an on-chain deposit path.
- All money mutations go through `CreditService` only.
- Ledger remains append-only source of truth; `Account.balance` remains a cache.
- Financial FKs remain on `billing.Account`, never `User`.
- Public HTTP surface remains under `/api/v1/billing/`.
- Domain events use snapshot payloads via the in-process bus ([ADR 005](./005-domain-events.md)).
- Architecture tests and idempotency follow ADR 012 mandatory invariants.

Extend the ADR 010 registry with a single ledger reference type:

```python
class LedgerReferenceType(models.TextChoices):
    # ... existing ...
    VOUCHER = "voucher", "Voucher"

REFERENCE_MODELS = {
    # ... existing ...
    LedgerReferenceType.VOUCHER: VoucherRedemption,
}
```

The ledger records only that credit came from a voucher redemption. Business meaning (promo vs gift, shared vs unique, issuer) lives on voucher models.

### Value vs rule: `RewardType`

Separate the **reward** from campaign **rules**.

```python
class RewardType(models.TextChoices):
    FIXED_CREDIT = "fixed_credit", "Fixed credit"
    PERCENT_BONUS = "percent_bonus", "Percent bonus"          # reserved
    FREE_SUBSCRIPTION = "free_subscription", "Free subscription"  # reserved
    FREE_TOPUP = "free_topup", "Free top-up"                  # reserved
```

- Campaign / Voucher carry `reward_type` plus type-specific fields.
- **v1 implements only `FIXED_CREDIT`** with `credit_amount` `Decimal(20,6)`.
- Other enum values are reserved so the model need not change later; v1 must not issue or redeem them (reject / not issued).

### Classification vs issuer

| Concern | Field | Values |
|---------|--------|--------|
| Business purpose | `VoucherType` | `PROMO`, `GIFT`, `ADMIN`, `PARTNER` |
| Who issued | `issued_by_type` | `SYSTEM`, `ADMIN`, `PARTNER`, `API` |
| Issuer reference | `issued_by_id` | nullable UUID/string (admin user, partner, API client) |

Future referral / affiliate / support flows map onto the same issuer types without new ledger reference types.

### `RedemptionMode`

```python
class RedemptionMode(models.TextChoices):
    SHARED = "shared", "Shared"
    UNIQUE = "unique", "Unique"
```

Both modes end in the same money path: validate → `VoucherRedemption` → `CreditService.credit(...)`.

### Schema (summary)

| Model | Role |
|-------|------|
| `VoucherCampaign` | Rules + optional shared code (`SUMMER2027`) |
| `VoucherBatch` | Generation unit for N unique codes (CSV/PDF re-download) |
| `Voucher` | Unique redeemable code (`RK-9H4K-7LQ2-XP8M`) |
| `VoucherRedemption` | Per-redeem audit row; **ledger `reference_id` anchor** |

#### `VoucherCampaign`

- `code` — shared code; unique; required when `redemption_mode=SHARED`
- `redemption_mode`, `voucher_type`, `reward_type`, `credit_amount` (when `FIXED_CREDIT`)
- Limits: `max_redemptions_total`, `max_redemptions_per_account`, `starts_at`, `expires_at`
- Status: `DRAFT` → `ACTIVE` → `EXPIRED` \| `REVOKED`
- `revoke_reason` nullable (status separate from reason), e.g. `fraud`, `expired_campaign`, `manual_revoke`, `duplicate_issue`
- Issuer fields on create
- User-segment targeting: **non-goal v1**
- **No physical DELETE** — status transitions only

#### `VoucherBatch`

```text
VoucherCampaign → VoucherBatch → N × Voucher (UNIQUE)
```

- Metadata: nullable `campaign` FK (standalone gift batches allowed), `size`, `created_at`, issuer fields, status
- Admin may re-download CSV/PDF for the same batch (implementation PR 2)
- Each batch voucher: unique `code`, `batch` FK, `campaign` FK when present

#### `Voucher`

- `code` unique
- `redemption_mode=UNIQUE`
- `voucher_type`, `reward_type`, `credit_amount` (v1), nullable `campaign` / `batch` FK
- Lifecycle: `CREATED` → `ACTIVE` → `REDEEMED` \| `EXPIRED` \| `REVOKED`
- `revoke_reason` nullable
- Gift vouchers: UNIQUE; often via batch; campaign FK optional
- **No physical DELETE**

#### `VoucherRedemption`

Every successful redeem (SHARED or UNIQUE) inserts one row:

- `account` FK → `billing.Account`
- nullable `voucher` / `campaign` FK — exactly one set (CHECK)
- `amount` (effectively credited), `ledger_entry_id`
- `redeemed_at`, `redeemed_ip`, `redeemed_user_agent`
- Limit enforcement via counts + row locks (and unique `(campaign, account)` when max per account is 1)
- **No physical DELETE** (append-only audit)

Ledger `reference_id` is **always** `VoucherRedemption.id`.

Suggested CHECKs (non-exhaustive):

| Model | Constraint |
|-------|------------|
| Campaign / Voucher | `credit_amount > 0` when `reward_type = fixed_credit` |
| Redemption | exactly one of `voucher_id`, `campaign_id` non-null |
| Campaign | `max_redemptions_total > 0` when set; `max_redemptions_per_account > 0` when set |

### Soft delete

Do **not** allow hard deletion of:

- `VoucherCampaign`
- `VoucherBatch`
- `Voucher`
- `VoucherRedemption`

Use status transitions (`ACTIVE` / `REVOKED` / `EXPIRED` / `REDEEMED`, etc.). This matches ADR 010’s append-only billing philosophy.

### Reserved codes

Case-insensitive denylist; never issue to users; reject on create, batch generation, and redeem resolution. Initial set:

```text
ADMIN, TEST, FREE, DEMO, NULL, SYSTEM
```

The set may grow via config without schema change.

### Code resolution

1. Normalize input; reject reserved codes.
2. Lookup `Voucher` by `code` where status `ACTIVE` and mode `UNIQUE`.
3. Else lookup `VoucherCampaign` by `code` where status `ACTIVE` and mode `SHARED`.
4. Else invalid code.
5. Codes must not collide across tables (enforcement strategy in implementation PR 1 — e.g. app check on create and/or a thin registry; ADR rule is absolute).

### Redeem path, concurrency, idempotency

**Mandatory lock order** (never reverse `Account` ↔ voucher/campaign):

```text
Voucher | VoucherCampaign   (code row, select_for_update)
    ↓
Account                     (select_for_update inside CreditService)
    ↓
Ledger                      (INSERT via CreditService)
```

Single DB transaction:

1. Lock campaign or voucher first.
2. Validate status, expiry, limits, `reward_type=FIXED_CREDIT`.
3. UNIQUE: transition `ACTIVE` → `REDEEMED`.
4. INSERT `VoucherRedemption`.
5. `CreditService.credit(...)` using the **same idempotency mechanism** as all other billing mutators (no special-case path). Example key: `voucher_redeem:{redemption.id}` via existing `idempotency_key`.
6. Publish `VoucherRedeemed` after success.

### Domain event

```python
VoucherRedeemed(
    event_version=1,
    voucher_id=...,       # nullable when SHARED campaign redeem
    campaign_id=...,      # nullable when UNIQUE voucher redeem
    redemption_id=...,
    account_id=...,
    amount=...,
    balance_after=...,
    ledger_entry_id=...,
    redeemed_at=...,
)
```

- Snapshot fields for handlers (analytics, email, referral, marketing) without re-fetching for the happy path.
- `event_version=1` allows a future `v2` payload without breaking existing handlers ([ADR 005](./005-domain-events.md) / ADR 010 event versioning pattern).

### API

```http
POST /api/v1/billing/vouchers/redeem
Authorization: required

{"code": "SUMMER-ABCD-1234"}

→ 200
{"credited": "25.000000", "balance": "120.500000"}
```

- Feature flag: `VOUCHERS_ENABLED` (alongside existing billing flags).
- When disabled: same pattern as other billing endpoints (404 / disabled behavior consistent with ADR 010).

### Architecture tests

Same boundary principle as ADR 010:

- Non-billing apps (`apps.orders`, `apps.airalo`, `apps.esims` / `apps.esim`, `apps.subscriptions`, and peers) **must not** import `Voucher*` models or write `Account.balance`.
- The only credit entry point remains `CreditService.credit(...)`.
- Prefer tests that reject hard-delete paths on voucher tables where enforceable.

## Financial invariants

All ADR 010 invariants apply. Additionally:

1. One ledger type only: `VOUCHER` → `VoucherRedemption`.
2. Redeem uses the standard billing `idempotency_key` path.
3. Lock order is always voucher/campaign → account → ledger.
4. No hard deletes of voucher entities.
5. v1 redeem grants only `FIXED_CREDIT` via `CreditService`.

## Non-goals (v1)

- Implementing `PERCENT_BONUS`, `FREE_SUBSCRIPTION`, or `FREE_TOPUP` (enum reserved only)
- Purchasing gift vouchers with USDT (admin/partner issuance only)
- Stripe / fiat
- User-segment targeting on campaigns
- Physical deletion of voucher entities
- Issuing or crediting outside `apps.billing` + `CreditService`
- Changing Faza 3 / ADR 010 prepaid USDT scope

## Consequences

### Positive

- Second credit source without redesigning billing ownership or ledger rules.
- SHARED + UNIQUE supported from day one without later schema forks.
- Batch + revoke reason + issuer audit support ops and fraud analysis.
- Explicit lock order reduces deadlock risk under concurrent redeems.
- Architecture tests keep Airalo/orders/subscriptions away from voucher internals.

### Negative

- More models than a “single promo code table” (`Campaign`, `Batch`, `Voucher`, `Redemption`).
- Cross-table code uniqueness needs careful enforcement in PR 1.
- Reserved `RewardType` values exist before product uses them (acceptable to avoid later migrations).

## Implementation note — PR roadmap (after Faza 3)

| PR | Repo | Scope |
|----|------|--------|
| **PR 1** | `roamkit-api` | Models + migrations + redeem service + `POST …/vouchers/redeem` + idempotency/concurrency tests + architecture tests + `LedgerReferenceType.VOUCHER` |
| **PR 2** | `roamkit-api` | Django admin: campaigns, batch generate + CSV/PDF re-download, revoke with `revoke_reason` |
| **PR 3** | `roamkit-web` (+ API read endpoints if needed) | Redeem UI, analytics/reporting on `VoucherRedeemed` |

Do **not** implement vouchers inside Faza 3. Faza 3 remains Polygon USDT prepaid credits only ([ADR 010](./010-polygon-usdt-prepaid-credits.md)).

**Gate before PR 1:** ADR 012 and this ADR are **Accepted**. Implementation must still pass the ADR 012 extension checklist and credit-source Definition of Done.

### Durability priorities (locked)

1. `VoucherBatch`
2. `RewardType` (v1 only `FIXED_CREDIT`)
3. Lock order `Voucher/Campaign → Account → Ledger`
4. Architecture test against bypassing `CreditService`
5. No physical deletes (status-only lifecycle)

## ADR 012 compliance

| Rule | Status |
|------|--------|
| CreditService only | ✅ |
| Ledger SoT | ✅ |
| Append-only ledger | ✅ |
| Account owner | ✅ |
| Idempotent | ✅ |
| Single DB transaction + lock | ✅ |
| Snapshot event | ✅ (`VoucherRedeemed`) |
| `/api/v1/billing/` only | ✅ (`POST …/vouchers/redeem`) |
| Feature flag / gate | ✅ (`VOUCHERS_ENABLED` — document in PR 1) |
| Architecture tests | ✅ (required in PR 1) |
| Does not revise ADR 010 invariants | ✅ |

## Related

- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — Polygon USDT prepaid credits (money path foundation)
- [ADR 012](./012-billing-extensibility-rules.md) — billing extensibility rules (must satisfy)
- [ADR 005](./005-domain-events.md) — in-process domain event bus
- [ADR_INDEX](../../ADR_INDEX.md)
