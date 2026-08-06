# ADR 019: Account pricing profiles

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-08 |
| Deciders | Solo operator (architecture lock before PR1–PR6) |

## Context

Some RoamKit accounts (e.g. family / private resellers) need a lower charge than the catalog list price, while public catalog stays at provider recommended retail. Price resolution must not become a second money path: [ADR 010](./010-polygon-usdt-prepaid-credits.md) still owns credits via `CreditService` and the append-only ledger.

Today `OrderService` / top-up flows debit live `Package.price_usd` with no per-account policy. Putting `discount_percent` on `Account` does not scale; re-resolving price after reserve creates race bugs when an admin edits a profile mid-fulfillment.

## Decision

Introduce a **pricing domain** (`apps.pricing`) separate from billing:

```text
Package (list + net)
        ↓
PricingContext → PricingService → PricingQuote
        ↓
Order / Topup snapshot (source of truth for charge)
        ↓
CreditService.debit(amount from snapshot only)
```

This ADR does **not** revise ADR 010 or [ADR 012](./012-billing-extensibility-rules.md). It does not add a credit source. It only defines how the **product charge amount** is chosen before spend.

### Architecture lock

```text
Architecture: LOCKED
Schema: Frozen after pricing schema PR (except bug fixes)
Execute implementation PRs sequentially (schema → service → debit → API → web → admin preview)
No new pricing features during that sequence
```

### `list_price` definition

**`list_price`** = provider **recommended retail / minimum selling price** as stored on catalog `Package.price_usd` (Airalo `recommended_retail_price` / `price` today).

It is **not** “whatever the customer sees”. The customer charge is `customer_price` (API field `price_usd` remains the chargeable amount for ADR 010 catalog presentation).

### Core types

- **`PricingProfile`** — named discount policy (`slug`, `discount_percent`, `floor_policy`, effective window, soft-delete via `archived_at`, optimistic `version`).
- **`Account.pricing_profile`** — nullable FK; many accounts share one profile.
- **`PricingContext`** — input to resolve (`account`, `list_price`, `net_price`, `order_type`, `timestamp`, optional preloaded profile). Extensible later (renewal, subscription, partner) without breaking the resolve signature.
- **`PricingQuote`** — frozen result including fingerprint: `pricing_profile_id`, `pricing_profile_version`, `pricing_context_hash`.
- **`PricingService.resolve(ctx) -> PricingQuote`** — sole price calculator. `CreditService` stays discount-agnostic.

### `discount_percent` semantics (normative)

**`discount_percent` represents the percentage of the partner’s margin that is given to the customer, not a percentage off the retail (list) price.**

Domain lock:

```text
0 <= discount_percent <= 100
```

### Customer price formula (wholesale path — normative)

Default path for new profiles (`floor_policy = wholesale`):

```text
L = list_price
N = net_price
D = discount_percent

margin = max(0, L − N)
C = N + margin × (100 − D) / 100
  = L − margin × D / 100
```

All arithmetic uses `Decimal`. **`money_round()` is applied only once, on the final `customer_price`.** No intermediate rounding.

By construction, when `N` is known and `L >= N`, `C >= N` (never sell below wholesale).

#### Worked examples

| L | N | D | Margin | C |
|--:|--:|--:|------:|--:|
| 25.00 | 5.00 | 50% | 20.00 | **15.00** |
| 25.00 | 5.00 | 100% | 20.00 | **5.00** (nabavna) |
| 57.00 | 50.00 | 5% | 7.00 | **56.65** |

The 5% example shows D applies to the **$7 margin**, not to retail ($57 × 0.95 = 54.15 would be wrong).

#### Invalid / missing catalog

These are **visible operational signals**, not silent business policy:

| Condition | Resolve | `pricing_reason` | Ops |
|-----------|---------|------------------|-----|
| `N` missing | `C = L` (retail) | `retail` | warning + metric `pricing.net_missing` |
| `L < N` | `C = L` (retail) | `invalid_margin` | warning + metric `pricing.invalid_margin` |

Repeated signals indicate a **catalog sync** problem, not a pricing-engine defect.

#### Interim note

PR2 initially implemented percent-off-list (`C = L × (100−D)/100` + optional wholesale floor). That interim arithmetic is **superseded** by this margin-share formula for the wholesale path.

### Floor policy

```text
NONE                    # legacy compatibility — percent-off-list; not recommended for new profiles
WHOLESALE               # v1 default — margin-share formula above
WHOLESALE_PLUS_MARGIN   # reserved
FIXED_MIN_MARGIN        # reserved
```

v1 implements only `NONE` and `WHOLESALE`. **`floor_policy=none` is a legacy compatibility mode and must not be used for new Family / partner profiles.**

### FormulaStrategy (future concept — not implemented)

ADR reserves a conceptual extension point (no code in v1):

```text
wholesale_share   # v1 default (margin-share formula)
legacy_percent    # floor_policy=none path
reserved          # WHOLESALE_PLUS_MARGIN / FIXED_MIN_MARGIN / future
```

v1 may keep the formula inline inside `PricingService`. Coupons, campaigns, loyalty remain future `PricingPolicy` list items (below).

### Debit-from-snapshot (normative)

1. Build `PricingContext`.
2. `quote = PricingService.resolve(ctx)` **once** at reserve.
3. Persist the full quote onto Order / Topup (`snapshot_schema_version`, profile id/version/slug, reasons, context hash, list + charged).
4. `CreditService.debit` / compensate / idempotent replay use **snapshotted charged amount only**.
5. **Never** re-resolve the profile for that order/topup after the snapshot is written.

### Migration note (formula amendment)

- **Existing Order / Topup snapshots remain authoritative.**
- There is **no data migration** of historical charged amounts.
- The margin-share formula applies only to **newly created quotes** after the engine change ships.
- Refunds, compensates, and idempotent replays for existing rows **must not** re-resolve with the new formula.

### Soft delete and concurrency

- No hard delete of profiles on product paths; set `archived_at` (+ inactive).
- At most one non-archived row per `slug` (partial unique constraint).
- Material field changes (`discount_percent`, `floor_policy`) auto-increment `version`; updates use optimistic `WHERE version = expected`.

### Feature flag

`PRICING_PROFILES_ENABLED` — when false, resolve always returns retail (list == customer). Deploy code dark, then enable.

### Public API shape (later PR)

Additive, non-breaking:

- `price_usd` — customer charge
- `list_price_usd` — provider list
- `discount_percent`, `pricing_reason`

Never expose `net_price`, margin, or quote fingerprint on public catalog. Internal preview may return fingerprint for ops.

### Preview

`POST /api/internal/pricing/preview` and Django admin preview both call **`PricingService.resolve` only**. Preview is **read-only**, idempotent, and must not write rows or emit durable side effects.

### Rounding

Single `money_round()` helper applied **once** to final `customer_price` (see formula section). No other `Decimal.quantize()` on pricing/spend paths.

### PricingPolicy (future)

ADR locks an extension point:

```text
PricingService → ordered PricingPolicy list → PricingQuote
```

Coupons, campaigns, loyalty, referral are future policies. v1 may inline profile + floor inside the service. See also FormulaStrategy (concept) above.

### Observability

Counters / histograms emitted from quote results only (**derived**, no histogram tables). Margin histograms are ops telemetry only.

Required counters for catalog integrity (formula amendment):

- `pricing.net_missing` — resolve fell back to retail because `net_price` was absent
- `pricing.invalid_margin` — resolve fell back to retail because `list_price < net_price`

### MSP note

Airalo partner terms treat recommended retail as a minimum selling price. Private account discounts below MSP are a **commercial/contract** risk, not a technical blocker. Product decision for private cohorts.

### Multi-provider

`PricingService` accepts base `list_price` / `net_price` from catalog — never provider DTOs (Airalo, etc.).

## Consequences

### Positive

- Shared profiles; audit fingerprints; no mid-fulfillment price drift; clear boundary vs ADR 010.
- Additive API preserves existing web `price_usd` contract.

### Negative / trade-offs

- Extra app and snapshot columns on Order/Topup.
- Billing `Account` gains a FK to `pricing` (cross-app dependency).

### Out of scope (this ADR / first ship)

- Coupon/campaign/loyalty policies
- `WHOLESALE_PLUS_MARGIN` / `FIXED_MIN_MARGIN` behavior
- B2B self-service portal
- Public net/margin exposure

## Compliance

| Rule | How satisfied |
|------|----------------|
| ADR 010 money path unchanged | Only debit amount source changes to snapshotted quote; still `CreditService` |
| ADR 012 | Not a credit source; no new ledger reference type required for pricing itself |
| Wholesale leak | `net_price` remains internal; public API forbidden fields unchanged |
