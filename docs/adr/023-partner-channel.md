# ADR 023: Partner Channel

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-10 |
| Amended | 2026-10-08 (user-id snapshots use the same scalar type as `User.id`; `source_id` uses each model's canonical primary key; grant body fallback is `400 invalid_request`; partner reads may send `X-Partner-Role` as a presentation hint; partner `email` is the full address and reads include `display_name`). 2026-10-08 invite amendment: many links per channel, `InviteVisit`, registration-bonus eligibility, and the consumer landing `/register?from=invite`. Money invariants in [ADR 010](./010-polygon-usdt-prepaid-credits.md) are unchanged; that ADR only gains `partner_invite_bonus`. |
| Deciders | Solo operator (architecture lock before schema / API) |
| Relates to | [ADR 010](./010-polygon-usdt-prepaid-credits.md), [ADR 012](./012-billing-extensibility-rules.md), [ADR 019](./019-account-pricing-profiles.md) (unchanged), [ADR 020](./020-organization-team-accounts.md) (narrow amendment only) |

## Context

RoamKit can attach a permanent invite link to one Organization so that new customers who arrive through that link are attributed to that partner. Attributed customers stay on their personal Accounts and pay through the existing purchase path. A share of RoamKit’s margin is credited to the Organization’s existing team Account. The partner sees those customers and that balance in a portal on `team.roamkit.net`, and may move credits from the team Account to a currently attributed customer, only up to the team balance.

[ADR 020](./020-organization-team-accounts.md) already has Organization, Membership, and exactly one team `billing.Account`. Its invite creates Membership, not a customer relationship. Its non-goal “B2B reseller portal” is narrowed by the amendment in that ADR: this document is that narrow channel, not a white-label store. Organization remains not the owner of money.

[ADR 019](./019-account-pricing-profiles.md) is unchanged. `discount_percent` is still the share of margin given to the buying account. It is not a partner commission.

No user id is hardcoded. Binding a specific operator and that operator’s existing customers is ops after the flag exists.

## Decision

Add a **Partner Channel** credit source that satisfies [ADR 012](./012-billing-extensibility-rules.md) and does not revise [ADR 019](./019-account-pricing-profiles.md). It does not change the money invariants in [ADR 010](./010-polygon-usdt-prepaid-credits.md). The only ADR 010 addition is the ledger type `partner_invite_bonus`, posted through the existing `CreditService`. Invite campaign rules in this ADR are the source of truth for visit attribution and registration-bonus eligibility.

```text
PartnerInviteLink
        ↓
CustomerAttribution          (personal User, not a Membership)
        ↓
personal Account debit       (existing order / topup / subscription path)
        ↓
PartnerMarginAccrual + CreditService.credit on the org team Account
        ↓
PartnerCreditGrant           (team Account → that customer’s personal Account)
```

```text
Architecture: LOCKED (Accepted)
Partner channel ≠ Membership and ≠ ADR 019 discount
Organization is not a money owner
PartnerChannel has no billing.Account FK
Money moves only through CreditService
ADR 019 is not amended
No schema or API before this ADR is Accepted
```

The customer charge stays on the existing pricing path. The partner channel does not call `PricingService` and does not read `discount_percent`. Commission is `PartnerChannel.revenue_share_percent`, calculated from the commercial snapshot’s `list_price` and `net_price`, not from the amount the customer was charged and not from a live catalog read.

`list_price` keeps the [ADR 019](./019-account-pricing-profiles.md) meaning: provider recommended retail stored as catalog `Package.price_usd`.

Deposits are not shared. There is no `partner_margin_reversal`, no `unrecovered_amount`, and no shortfall lock. A later manual margin return, if support needs one, is a separate decision.

### Ledger reference types

Added in the schema migration, before any writer exists:

| `reference_type` | `reference_id` | `REFERENCE_MODELS` |
|------------------|----------------|--------------------|
| `partner_margin` | `PartnerMarginAccrual` id | `PartnerMarginAccrual` |
| `partner_grant_out` | `PartnerCreditGrant` id | `PartnerCreditGrant` |
| `partner_grant_in` | `PartnerCreditGrant` id | `PartnerCreditGrant` |
| `partner_invite_bonus` | `CustomerAttribution` id | `CustomerAttribution` |

`partner_invite_bonus` is the platform-funded registration bonus. Idempotency key is `invite-registration-bonus:<user_id>`. Marketing rules for when that row is written are below, not in ADR 010.

`LedgerReferenceType` stays a `CharField` (`max_length` 32). Reverse of that migration must not delete ledger rows. If reverse would narrow allowed values, it stops when any row uses one of these types.

## Invariants

| # | Invariant |
|---|-----------|
| 1 | `CreditService` is the only mutator of `Account.balance` and the only inserter of these ledger rows. |
| 2 | Ledger is source of truth. `Account.balance` is a cache. Partner UI must not recompute balance as margins + admin adjustments − grants. |
| 3 | `PartnerChannel` has no `account` FK and no team-account snapshot. Money path is `PartnerChannel` → `Organization` → the existing team `billing.Account` from ADR 020. |
| 4 | Customers are not memberships. They keep personal Accounts. The channel does not spend the team wallet on their behalf. |
| 5 | Accrual, grant, attribution history, and renewal-cycle rows are immutable as specified below. Ledger stays append-only. |
| 6 | One user has at most one current `CustomerAttribution`. The user flow never retargets it. |
| 7 | `PartnerChannel.is_active` controls only new accruals. `PartnerInviteLink.is_active` controls only new attributions. They do not change each other. |
| 8 | A missing team Account is a system error `partner_channel.team_account_missing`, not a business skip and not `insufficient_funds`. |
| 9 | `PARTNER_CHANNEL_ENABLED` defaults off. The server checks it on portal reads, portal writes, and the margin path. Hiding UI is not the check. |
| 10 | Public partner responses do not include `net_price`, `margin`, `revenue_share_percent`, or `team_account_id`. Partner `email` is the full address. `display_name` is present and may be empty. |

`revenue_share_percent` is `Decimal(5,2)` with `CHECK 0 <= revenue_share_percent <= 100`. Money fields on the accrual are `Decimal(20,6)`. Arithmetic is `Decimal`, never `float`. The portal does not compute `partner_share`.

### PartnerChannel lifecycle

`PartnerChannel` is 1:1 with `Organization` (`organization` PROTECT). It is not created with the Organization. An admin service creates it only when that Organization already has a team Account, in one `transaction.atomic()`:

```text
1. check Organization + team Account
2. create PartnerChannel (starts active)
3. generate a cryptographic token
4. create PartnerInviteLink
5. commit
```

Token is URL-safe, not sequential, does not encode `organization_id`, at least 128 bits (`secrets.token_urlsafe(24)` or equivalent). `UNIQUE(token)` is global. Collision retries a new token inside the same transaction. A channel without a link does not commit.

That first link is the canonical portal link. Its classification is empty and its bonus is zero:

```text
name = ""
bonus_amount = 0.000000
source = ""
campaign = ""
content = ""
```

A channel may later have more links. The canonical link is the one with the smallest `(created_at, id)`. The portal still shows only that link. Additional campaign links are created in Django admin, not in the portal.

The channel is never physically deleted, even with no children. No admin delete, no delete service, the model refuses delete. Shutdown is `is_active=False` only. An Organization that has a channel cannot be deleted. Formal archival is out of v1.

`is_active` and `revenue_share_percent` are not saved by a raw or inline edit. Both go through a service that `SELECT … FOR UPDATE` locks that `PartnerChannel` row only, the same row the margin path locks. It does not lock attribution or the team Account.

`is_active=False` stops new accruals only. Existing team balance may still be granted. The portal still does not see `net_price` or `margin`.

### Feature flag off

`PARTNER_CHANNEL_ENABLED=false` creates no new `InviteVisit`, no new pending row, no new attribution, no new accrual, and no team credit. Portal and grant return `404` with `{"code":"partner_channel_disabled"}`. Public `GET /join/<token>` stays a generic 404 with no JSON and no cookie. Fulfillment still succeeds. The margin path logs info `partner_margin.flag_disabled`. Existing rows and balances are unchanged. Django admin still shows them.

### User deletion

No `CASCADE` from `User` onto accrual, grant, ledger, or `CustomerAttributionHistory`. Those FKs are `SET_NULL` plus a user-id snapshot and no email:

- accrual and grant: `customer_user_id_snapshot`
- grant: `granted_by_user_id_snapshot`
- history: `changed_by_user_id_snapshot`

User identity snapshots use the same scalar type as `User.id` (currently bigint), not UUID. They store only the immutable user identifier snapshot, never email or PII.

`granted_by` becomes `NULL` only when the User is deleted, not when Membership is revoked. `CustomerAttribution.user` stays `PROTECT`. This ADR does not add an anonymization product.

### Membership vs channel

Membership status affects only future portal access and write rights. It does not change `CustomerAttribution`, `PartnerMarginAccrual`, `PartnerCreditGrant`, or team balance. `revoked` and `suspended` lose the portal immediately. A role change applies on the next request. The browser must not cache the role as authority. Existing grants remain.

The channel does not require an active owner. [ADR 020](./020-organization-team-accounts.md) membership cardinality is unchanged. If no active owner is present, admin and viewer keep their own rights, including grant for admin. Nobody can regenerate, activate, or deactivate the link until an owner exists. The backend does not promote an admin. The link stays as it was.

## Attribution

One `CustomerAttribution` per user (`user` unique, PROTECT). `invite_visit` is the converting click. That FK is not unique, so the visit is the click record rather than a one-user token. `invite_token` is the token copied at insert. It is not the live link and it is not the authority for a later attribution.

Fields copied at insert and frozen afterwards:

```text
invite_token
invite_name_snapshot
invite_source_snapshot
invite_campaign_snapshot
invite_content_snapshot
utm_source_snapshot
utm_medium_snapshot
utm_campaign_snapshot
utm_content_snapshot
registered_via_invite
bonus_amount_snapshot
invite_visit
```

`registered_via_invite = true` means the User row was inserted by an invite registration (email `CREATED` confirmed, or Google `CREATED`). `false` means an existing user was only attached, or an admin assigned the customer.

`bonus_amount_snapshot`:

```text
NULL       not eligible for a registration bonus
0.000000   new invite registration through a zero-bonus link
> 0        new invite registration and the exact bonus for that row
```

`NULL` is not the same as zero. Zero is eligible and still writes no ledger row. A positive snapshot is the amount credited.

A later visit does not change an existing row. Channel deactivation does not delete it. Reactivation resumes commission for the same partner. The first admin bind is `source=admin`, `registered_via_invite=false`, `bonus_amount_snapshot=NULL`, `invite_visit=NULL`, and empty invite snapshots. It writes no history row and no bonus.

Django admin has no form that saves `CustomerAttribution` or edits `partner_channel`. Two service actions only:

| Action | Rule |
|--------|------|
| Assign customer to partner | User + channel, `source=admin`, actor required. Rejected if an attribution exists. No history row. Actor stays in the Django admin log. |
| Transfer customer to another partner | New channel. Only `transfer_customer_attribution()`. Rejected if no attribution exists. Changes only `partner_channel`. Writes no `CustomerAttributionHistory` row. |

`transfer_customer_attribution` locks the attribution row. On that row it changes only `partner_channel`. It does not change `invite_visit`, `invite_token`, `source`, the invite or UTM snapshots, `registered_via_invite`, or `bonus_amount_snapshot`. It does not credit a second registration bonus and it does not call `CreditService`. It does not write a `CustomerAttributionHistory` row. This feature does not add that record. The same channel is a no-op. A missing attribution raises `CustomerAttribution.DoesNotExist` and does not insert a row. It does not move old accruals or team balance. Later purchases follow the new partner.

Consume, email confirmation, and Google auth do not call `transfer_customer_attribution`. A click on another team's link does not move the customer.

### Pending row

`PendingPartnerAttribution`: OneToOne `user`, `partner_channel`, required `invite_token_snapshot`, nullable `invite_visit` (PROTECT), `expires_at`, `created_at`. Index `(expires_at)`. Only for the inactive user created by email register. One pending per user. New rows always store `invite_visit`. Snapshots and `bonus_amount_snapshot` are not written on this row.

Two clocks, not one:

```text
30 days   click attribution window, from InviteVisit.created_at
24 hours  PendingPartnerAttribution, from email submit
```

Email submit validates the visit with the 30-day window. A submit on day 29 still creates a pending row. Confirmation does not measure the click again. It accepts the pending row while `expires_at` is inside its 24 hours, even if the click has by then passed 30 days. A click that is already older than 30 days at submit creates the account and no pending row. Opening a link and registering two months later is not attributed from that click.

`expires_at` is `now + 24 hours` at submit. It is not the cookie `issued_at` and it is not 30 days. `activate` checks `expires_at > now()` first and deletes only that expired row, even if the periodic job has not run. An inactive link, or `visit.created_at < invite_link.regenerated_at`, deletes the pending row and creates no attribution and no bonus. Successful attribution deletes the pending row in the same transaction. If the user already has an attribution, the pending row is discarded. Auth does not bulk-clean other rows.

`cleanup_expired_partner_attributions` deletes only `PendingPartnerAttribution` where `expires_at <= now()`. It does not touch attribution, history, visits, accruals, or ledger. A late job does not change activate’s outcome. Expiry does not delete `InviteVisit`.

A pending row that has no `invite_visit` is legacy. Confirmation of that row still compares `invite_token_snapshot` with the canonical link, writes no invite snapshots, and writes no bonus. New code does not create that shape. The converting click, not the canonical token, is the source of a current attribution.

## Margin and accrual

Accrual is created only when an order or topup becomes `fulfilled`, or a subscription renewal becomes `renewed`. Not on `order_created`, `payment_started`, `topup_requested`, or `subscription_due`.

Order and topup debit the customer before the provider call, in a separate transaction. Provider failure still refunds the customer and does not create an accrual, so this channel has no partner reversal. The accrual is in the same transaction as `_persist_fulfillment`. If that transaction rolls back, there is no `fulfilled` and no accrual.

Subscription success is `renew_one` outcome `renewed`: debit and the billing-period move in one transaction, and the accrual with them. `paused` for insufficient funds creates no accrual.

### Prices come from the snapshot

The margin service does not read `Package.price_usd` or a live provider `net_price` at fulfillment.

| Source | Snapshot |
|--------|----------|
| Order | Existing `list_price_usd` and `net_price_usd`. A later catalog change does not change them. |
| Topup | Existing `list_price_usd` plus new `Topup.net_price_usd`: `Decimal(20,6)`, nullable, no backfill. |
| Subscription | The renewal-cycle row below. `Subscription.price_per_period` is not that snapshot. |

`Topup.net_price_usd` is written once at topup creation, before the provider call. Fulfillment and retry do not write it. Django admin shows it read-only. Historical `NULL` stays `NULL`. `NULL` is skip `partner_margin.net_missing`. Old topups get no retroactive accrual.

Renewal cycle, one durable row, not prices held only in task memory:

```text
subscription_id
billing_date                 pure date, equal to next_billing_date read before the attempt
renewal_list_price_usd       Decimal(20,6), write-once
renewal_net_price_usd        Decimal(20,6), write-once
status                       pending | renewed | paused | failed
created_at
```

`UNIQUE (subscription_id, billing_date)`. A second partial unique index is `UNIQUE (subscription_id) WHERE status IN ('pending', 'paused', 'failed')`. The pair alone does not stop two unfinished cycles on different dates. `renewed` is outside that partial index, so older finished cycles remain. Prices are write-once. The row is committed before debit, so a failed debit does not delete it. Retry of the same pair does not create a new snapshot. Debit and accrual both read that row. `partner-margin:subscription:<subscription_id>:<billing_date>` stays tied to it. Missing net is only `partner_margin.net_missing`.

`pending` is created with the snapshot. `renewed` only after a successful debit and the billing-period move. `paused` means insufficient funds and no accrual. `failed` is only a deliberately stored technical error. A rolled-back attempt leaves `pending`, not a new `failed`. Status changes do not change prices. `renewed` is terminal. The service may change only `status`, and not once `renewed`. Django admin shows the row read-only and has no delete. The model refuses deletion. FK to `Subscription` is `PROTECT`.

While the cycle for the current `billing_date` is not `renewed`, `Subscription.next_billing_date` does not move and no next-period cycle is created. The partial unique index enforces at most one unfinished cycle (`pending`, `paused`, or `failed`) per subscription. `billing_date` is a pure date. No timezone conversion. The idempotency string is `YYYY-MM-DD` from that DB date. Success sets `next_billing_date = billing_date + 30 days`, not `now() + 30 days`. A cycle dated `2026-10-08` that succeeds on `2026-10-15` gets `2026-11-07`. The same transaction commits `renewed`, the debit, the accrual if any, and the date move.

### Formula

After validation (`net` present, `net >= 0`, `list >= net`, percent > 0):

```text
margin = list_price - net_price
partner_share = round_6(margin * revenue_share_percent / 100)
```

`list < net` is a skip before the formula, not `max(0, …)`. `margin` is not rounded on its own. Rounding is once, on `partner_share`, 6 decimal places, `ROUND_HALF_UP`, quantum `0.000001`, the same quantum `CreditService` already uses. Pricing `money_round()` (0.01) is not used. The snapshot stores the values actually used.

Example: L=`25.000000`, N=`5.000000`, 33.33% → margin `20.000000`, `partner_share` `6.666000`. L=`25`, N=`5`, 50% → share `10`. Tests include `33.33%`, `12.50%`, a very small margin, and a value on the sixth-decimal boundary.

### Skip vs rollback

No accrual and no team credit. Fulfillment stays `fulfilled` / `renewed`. Info log:

| Reason | When |
|--------|------|
| `partner_margin.flag_disabled` | Flag off |
| `partner_margin.no_attribution` | No attribution |
| `partner_margin.channel_inactive` | Channel `is_active` false |
| `partner_margin.net_missing` | Net missing |
| `partner_margin.net_negative` | Net < 0 |
| `partner_margin.invalid_margin` | List < net |
| `partner_margin.zero_share` | Percent 0 or rounded share 0 |

`CreditService` or DB failure, and `partner_channel.team_account_missing`, are errors. The fulfillment transaction rolls back. A grant writes nothing and does not return `insufficient_funds`.

Every margin attempt logs `source_type`, `source_id`, `customer_user_id`, `partner_channel_id`, `organization_id`, and `reason` (`null` when absent). Success is info `reason=accrued` plus `partner_share`, `ledger_entry_id`, and `accrual_id`. v1 adds no new in-process domain event. The accrual row is the snapshot.

### Accrual row

`PartnerMarginAccrual` remembers the relationship at fulfillment. Stats never re-read the current attribution.

```text
partner_channel
customer_user                  SET_NULL
customer_user_id_snapshot
customer_attribution           nullable audit FK
source_type                    order | topup | subscription
source_id
list_price
net_price
margin
revenue_share_percent
partner_share
ledger_entry
created_at
```

`source_id` is built by one helper. The helper writes the canonical string of that model's actual primary-key type. It does not assume every primary key is a UUID. Date is `YYYY-MM-DD`.

```text
order        → canonical Order.id
topup        → canonical internal Topup.id     not external_order_id
subscription → canonical Subscription.id:<billing_date>
```

`billing_date` is the cycle date read before `next_billing_date` moves. Idempotency key is only `partner-margin:{source_type}:{source_id}`. `source_id` is immutable after insert.

Inside the `fulfilled` / `renewed` transaction:

```text
PartnerMarginAccrual exists
⇔
exactly one partner_margin ledger entry exists
```

The accrual id is generated before `CreditService.credit`. `reference_type=partner_margin`, `reference_id` is that id, not the order, topup, or subscription id. `UNIQUE (source_type, source_id)` and `UNIQUE (ledger_entry)`. A second fulfillment returns the same pair.

Stats filter `accrual.partner_channel` and `customer_user`. Example: X earns 10 for A on 01.10, transfers to B on 05.10, earns 7 for B on 10.10 → A=10, B=7.

v1 has no public accrual list, no `/earnings`, and no customer-detail accrual route. Accruals feed only summary and customers. `source_type` and `source_id` stay in the database for audit and Django admin. A later endpoint must not change this model. The row cannot be updated or deleted by the service or admin.

Indexes: `(partner_channel, created_at)`, `(partner_channel, customer_user)`, `(customer_user)`.

### Lock order on fulfillment

Subscription renewal already holds `SELECT … FOR UPDATE` on the renewal-cycle row and keeps it until `renewed` commits. Then:

1. `SELECT … FOR UPDATE` `CustomerAttribution`. Partner and customer for the snapshot come only from that locked row.
2. `SELECT … FOR UPDATE` that `PartnerChannel`, then read `is_active` and percent.
3. If the source pair exists, return it. Otherwise generate the accrual id. `CreditService.credit` locks the team Account.
4. Insert the accrual with that id and `ledger_entry`. Commit with `fulfilled` / `renewed`.

Whoever commits first wins between transfer and fulfillment. There is no read of partner A, then a transfer to B, then a credit using the stale read. The same channel lock covers an admin `is_active` or percent change. A deactivation or percent change committed before fulfillment uses the new value. Fulfillment committed first keeps the old value.

Two renewal workers lock the cycle row before debit and before any status change. If the first commits `renewed`, the second does nothing. If the first leaves `paused` or `failed`, the second may retry the same row and the same snapshot. At most one debit and one partner accrual per cycle.

## Grants

`PartnerCreditGrant` is not tied to individual accruals. The partner may spend margin credits and `admin_adjustment` credits together. `SUM(partner_share)` may differ from `Account.balance`.

Direction is only that channel’s team Account → the personal Account of a customer whose current attribution is that channel. No arbitrary user, no team-to-team, no cash-out. `is_active=False` does not block a grant of existing balance.

Owner or admin only, from the active Membership loaded on the server. Amount is `Decimal(20,6)`, strictly `> 0` and `<=` team balance. If balance is already negative, reject the grant, write an error log, and do not rewrite balance to 0. `InsufficientFundsError` still prevents a negative result. The partner UI shows `max(0, balance)`. `partner_channel_reconcile` does not repair that balance and does not set it equal to the sum of accruals.

One transaction. Lock order: `CustomerAttribution`, then team Account, then the customer’s personal Account. If under that lock the customer is no longer on this channel, write nothing. Both money steps go through `CreditService`. The grant UUID is generated before the two calls so both ledger rows share `reference_id` = grant id.

| Leg | Account | Delta | `reference_type` |
|-----|---------|-------|------------------|
| Debit | Team | `-amount` | `partner_grant_out` |
| Credit | Customer personal | `+amount` | `partner_grant_in` |

The grant row stores `partner_channel`, `customer_user`, `customer_attribution`, `granted_by`, `amount`, `idempotency_key`, `debit_ledger_entry`, `credit_ledger_entry`, `created_at`. Unique `(partner_channel, idempotency_key)`, unique debit ledger, unique credit ledger. Immutable. No delete.

`POST /api/v1/billing/partner-grants/` body is only `customer_id`, `amount` (`"10.000000"`), and `idempotency_key`. Success returns only `grant_id`, `customer_id`, `amount` as a 6dp string, and `created_at` ISO-8601 UTC. No team balance. The portal refetches summary.

Identical replay returns the original grant even if the caller is rate-limited. The same key with a different customer or amount returns `409 idempotency_key_conflict` and does not return the old grant as success.

## Invite links, visits, and registration bonus

A channel has many reusable `PartnerInviteLink` rows. Links are not email-bound and not single-use. A link has no `expires_at`. It has no `utm_medium`, QR payload, click limit, campaign date range, currency, or `created_by`.

```text
id
partner_channel          PROTECT, not unique
token                    globally UNIQUE, editable only by regenerate
name
bonus_amount             Decimal(20,6), CHECK >= 0
source
campaign
content
is_active
created_at
updated_at
regenerated_at           nullable
```

`source`, `campaign`, and `content` are admin classification of the link. They are not copied from the request. Channel shutdown does not flip `link.is_active`. The link cannot be deleted. `InviteVisit` and `CustomerAttribution.invite_link` are PROTECT, so a link with history cannot be removed.

After the first `InviteVisit`, `save` and `QuerySet.update` reject changes to `partner_channel`, `source`, `campaign`, and `content`. Still writable: `name`, `bonus_amount`, `is_active`. Token changes only through regenerate.

Django admin creates the extra campaign links and may edit `name`, `bonus_amount`, and `is_active`. `token` is readonly. After the first visit the frozen fields are readonly in that form as well. A bonus change made through Django admin stays in the Django admin log. There is no portal editor for bonus, classification, or campaign links.

The portal invite card and `GET /api/v1/orgs/partner/invite-link/` still return only the canonical link: `url`, `is_active`, `created_at`, `regenerated_at`. Owner regenerate, activate, and deactivate lock that canonical row. They do not list or edit campaign links.

### InviteVisit

One append-only row per valid `GET /join/<token>`. It stays after the attribution window expires. Deleting it is refused.

```text
id              UUID
invite_link     FK PROTECT
utm_source
utm_medium
utm_campaign
utm_content
created_at
```

Index `(invite_link, created_at)`, name `bill_invite_visit_link_at`.

UTM is the request, not the link classification. Web reads `searchParams.getAll(name)[0]`.

```text
missing or empty     → ""
longer than 128      → truncate
duplicate parameter  → first value
no lowercase, trim, or campaign cleanup
```

Not stored: IP, User-Agent, Referer, language, country, user id. The visit has no user FK. Do not infer `source` from Referer.

Join and regenerate both `SELECT … FOR UPDATE` the `PartnerInviteLink` row they act on, so a join of a token and a regenerate of that same link serialize. Regenerate, in that transaction, sets a new token and `regenerated_at`. The old token is then a generic 404. Visits are not deleted. `CustomerAttribution` is not updated. Pending rows that stored the old token are deleted. A visit with `visit.created_at < invite_link.regenerated_at` cannot create a new attribution. An existing attribution is not revalidated against that visit.

`GET /join/<token>` for an unknown, inactive, or regenerated-away token returns the same generic 404. No cookie, no visit, no attribution, no ledger. Logs must not contain the full token or join URL. Optional fingerprint: `sha256(token)[0:12]`. Proxy access logs may still see the path. Redacting them is out of scope. The route does not embed the token in HTML or JS. `Cache-Control: no-store`.

| Channel | Link | Result |
|---------|------|--------|
| Off | On | New customers can still bind through an active link. Purchases earn nothing until the channel is active. |
| On | Off | That link creates no visit and no new attribution. Existing attributed customers still earn. |

### Cookie

`partner_pending` is host-only on the consumer host (`roamkit.net`, and the same on `staging.roamkit.net`): `HttpOnly`, `Secure` when the public origin is `https`, `SameSite=Lax`, `Path=/`, `Max-Age` 30 days, no `Domain=.roamkit.net`. `api.roamkit.net` and `team.roamkit.net` do not receive it. JavaScript cannot read it. The web app does not parse `visit_id`.

The signed payload is only:

```text
visit_id
issued_at
```

Signer salt is `roamkit.partner-pending`. Signature max age is 30 days from `issued_at`. That clock is not the business window. The business window is 30 days from `InviteVisit.created_at`. An expired cookie or an expired window does not delete the visit.

The last valid `/join/<token>` replaces the cookie. An invalid join does not clear an existing cookie. UTM is stored on the visit at that request. The redirect does not carry the token or `utm_*`.

```text
GET /join/<token>?utm_source=tiktok
→ InviteVisit
→ signed partner_pending
→ redirect /register?from=invite
```

`from=invite` is a UX label. It is not proof that a visit exists. The cookie, the visit, and the backend are the authority.

Email submit does not clear the cookie. Clearing it there would distinguish a new address from an existing one. Ordinary consumer logout does not clear it. A failed Google sign-in does not clear it.

The cookie may be cleared after a terminal success: a successful Google auth response that had a cookie, or a successful authenticated consume (`created`, `noop`, or `ignored`). An invalid or expired cookie may be cleared on that same terminal path. `401` and `5xx` on consume leave the cookie.

### Email registration

The browser posts to the same-origin web route `POST /api/auth/register`. That route reads `partner_pending` and, when present, forwards it as `X-Partner-Pending`. It does not put `visit_id` in the body and it does not clear the cookie.

`register_user` returns `CREATED` only when that call inserted the User, otherwise `EXISTING`. The public response is the same either way. A pending row is created only for `CREATED` plus a visit that passes the 30-day check. An invalid invite still creates the account and creates no pending row. It does not fail registration.

Confirmation reads the pending row, not the cookie. It does not re-check the 30-day click window. It does check that the link is active and that the visit is not older than `regenerated_at`. Attribution, snapshots, and `bonus_amount_snapshot` are written then, from the visit's link as it is at confirmation, and the pending row is deleted in that same transaction. The campaign link on the visit is used. The canonical link is not substituted.

`bonus_amount` may change while the pending row is open. Confirmation reads the current amount. An attribution that already exists keeps the snapshot it was inserted with.

### Google

Google stays a GIS popup. The credential is not an OAuth redirect and the Google URL does not carry the invite. The browser posts the credential to the same-origin web route:

```text
POST /api/auth/google
```

The web route reads `partner_pending` on the server and forwards the credential to the API. When the cookie is present it sends `X-Partner-Pending`. The body stays `{credential}`. This browser call does not depend on CORS to `api.roamkit.net`. An invalid, expired, inactive, or regenerated invite does not fail Google auth. The web app does not decide whether the invite is valid, and it does not decide `CREATED`, `EXISTING`, or `LINKED` in order to grant a bonus.

```text
CREATED + valid visit  → registration attribution, eligible for the bonus
EXISTING / LINKED      → consume, not eligible for the registration bonus
```

If an active user already has `PendingPartnerAttribution`, consume does not insert an attribution and does not delete that pending row. Email confirmation keeps priority. An inactive email user is still activated through the pending row before consume.

After a successful Google auth the browser goes to its normal next page. It does not also open `/join/complete`. The API already attributed or consumed while the header was present. A second consume would be a no-op, not a second attribution, and the client does not start one.

### Already signed-in visitor

`GET /join/<token>` cannot see the Bearer in `localStorage` and does not attribute. A valid join still redirects to `/register?from=invite`.

An authenticated visitor who then opens `/register?from=invite` is sent to `/join/complete`. A normal `/register` without that label still sends an authenticated visitor to `/me/esims`.

`/join/complete` is the consume page for that already signed-in visitor. Without a Bearer it asks the visitor to sign in again. With a Bearer the browser calls only:

```text
POST https://roamkit.net/join/complete/consume
Authorization: Bearer <user-token>
```

The same route exists on `staging.roamkit.net`. It is not on the team host. The browser does not send `partner_channel_id`, the invite token, `visit_id`, or an organization id, and never sees the cookie body.

Next.js is not the authority for user id or partner id. It forwards the Bearer and the signed cookie to the API. The API verifies the signature. A local Next.js check is not enough. The API does not accept a plain `user_id` or `partner_channel_id` as proof. This endpoint is not a public partner route and is not in the public partner OpenAPI schema.

The API sets `request.user` from the Bearer and reads `visit_id` from the signed payload.

```text
no attribution, no pending, valid visit
→ CustomerAttribution
→ registered_via_invite = false
→ bonus_amount_snapshot = NULL
→ {"status":"created"}

attribution already exists
→ no-op, row unchanged, no transfer
→ {"status":"noop"}

pending row exists, or visit invalid / inactive / pre-regeneration / flag off
→ no new attribution
→ {"status":"ignored"}
```

`ignored` does not say which of those cases it was, and it does not delete an email pending row. Two parallel first consumes: one inserts, the other returns `noop`, not an integrity error. HTTP 200 with any of the three statuses is a finished consume. The page then goes to `/me/esims`. It does not stay on an error because the cookie is stale, the user is already attributed, or the visit was ignored. `401` stays a sign-in message and keeps the cookie. `5xx` keeps the cookie and shows a retry.

The body on 200 is exactly `{"status":"created"|"noop"|"ignored"}`. No partner id, token, organization id, or reason.

### Registration bonus

The platform pays the bonus through `CreditService` onto the new user's personal Account. The team Account balance does not change. There is no `CreditGranted` event for this credit.

A ledger row is inserted only when `registered_via_invite` is true and `bonus_amount_snapshot > 0`. `NULL` and `0.000000` do not call `CreditService` and do not insert a ledger row.

```text
reference_type   = partner_invite_bonus
reference_id     = CustomerAttribution.id
idempotency_key  = invite-registration-bonus:<user_id>
```

One user receives this bonus at most once. The amount is `bonus_amount` on the visit's link at attribution insert, not the amount at the click. The credit runs in the same database transaction as that insert. If `CreditService` fails, the attribution insert rolls back with it: email activation stays uncommitted, and a Google `CREATED` user row is not kept. Consume, `EXISTING`, `LINKED`, admin assign, legacy pending confirmation, and transfer never credit this bonus.

### Team host auth

`localStorage` is per origin. The token on `roamkit.net` is not the token on `team.roamkit.net`. No copy and no `postMessage`.

Login starts and ends on the same team host. Google stays GIS `ux_mode: "popup"`. The in-page callback posts the credential to the same-origin web route `POST /api/auth/google`, which forwards it to the API. No redirect mode and no OAuth redirect to `roamkit.net`. The team host does not hold `partner_pending`. Authorized JavaScript origins gain exactly `https://team.roamkit.net` and `https://team.staging.roamkit.net`. No wildcard.

`next` on the team host may be only `/`, `/customers`, or `/grants`. Anything else, including `/me/esims` and absolute URLs, falls back to `/`.

Logout on `team.roamkit.net` deletes only that origin’s Bearer and returns to `/login` on that host. Staging deletes only the staging team token. Neither touches consumer `localStorage` nor `partner_pending`. Consumer logout does not touch the team token.

`401` from a portal call (`summary`, `customers`, `grants`, invite-link, regenerate, activate, deactivate, `partner-grants`) deletes that team Bearer and goes to `/login?next=<current allowed route>`. A `401` from the login form itself stays on the form.

## Portal and API

The shell is the existing Next.js app. Middleware reads `Host` before HTML.

```text
roamkit.net | www.roamkit.net | staging.roamkit.net → consumer web
team.roamkit.net | team.staging.roamkit.net         → partner shell
```

No public `roamkit.net/team`. No query flag selects the team shell. `from=invite` on the consumer register page is only the invite label defined above. v1 team routes are `/`, `/customers`, and `/grants`. `/` is the dashboard: `total_earned`, `available_balance`, `accrual_counts`, and one invite card. Unauthenticated users go to `/login` on that same host.

Portal roles are the active ADR 020 Membership. The backend loads the role.

| Role | Access |
|------|--------|
| `viewer` | Customers, summary, team balance, grant history, invite link. No grant. No link writes. |
| `admin` | Viewer plus grant. |
| `owner` | Admin plus regenerate and link activate/deactivate. |
| `member`, `suspended`, `revoked` | No portal. |

`revenue_share_percent` and `PartnerChannel.is_active` are Django admin only.

Tenant resolution never trusts `organization_id`, `partner_channel_id`, or `team_account_id`:

```text
request.user
→ exactly one active owner/admin/viewer Membership
→ Organization that has a PartnerChannel
→ that channel’s team Account
→ current attributions inside that channel
```

Zero matches: `403 partner_access_denied`. More than one: `409 partner_context_ambiguous` with only that code. No org picker. A foreign `customer_id` returns no data and receives no grant. Read URLs contain no org or channel id.

### Error envelope

Every portal and grant error body is exactly `{"code":"..."}`. No `detail`, field names, org ids, or serializer output. Check order: authentication, flag, membership, action permission, resource.

```text
401 authentication_required
403 partner_access_denied
403 partner_grant_forbidden
403 partner_invite_forbidden
409 partner_context_ambiguous
404 partner_channel_disabled
404 customer_not_found
409 customer_attribution_changed
409 insufficient_funds
409 idempotency_key_conflict
429 rate_limited
400 invalid_request        grant body that is malformed or semantically invalid and is not exclusively invalid_amount
400 invalid_amount
400 invalid_page | invalid_page_size | invalid_sort | invalid_order
400 invalid_query          customers only, q longer than 254 after trim
```

`invalid_amount` stays the narrower grant case: the body is otherwise valid and only `amount` is not. Any other malformed or semantically invalid grant body, including a bad `amount` together with another body problem, is `invalid_request`.

`customer_not_found` must not reveal that the user id exists at another partner. `customer_attribution_changed` means the customer was transferred off this channel under the attribution lock. A valid page past the end is not an error: return `count`, the requested `page`, `page_size`, and `results: []`.

`401` on the team host is the auth cleanup above, not a portal error screen. `403`, `404 partner_channel_disabled`, and `409 partner_context_ambiguous` replace the page with the portal screen (access denied, feature unavailable, or contact support with no org list). Network and `5xx` stay on that block with retry.

### Read contracts

`Cache-Control: no-store` on the whole Partner Channel HTTP surface, including errors and `POST /join/complete/consume`.

A successful partner read (`summary`, `customers`, `grants`, `invite-link`) may include `X-Partner-Role` with only `owner`, `admin`, or `viewer` as a presentation hint for the portal. It is not authorization, it does not affect tenant resolution, and it is absent from errors, writes, and consumer endpoints. Team origins expose it with `Access-Control-Expose-Headers: X-Partner-Role`. Backend role checks stay authoritative for every write.

`GET /api/v1/orgs/partner/summary` is the only dashboard numbers endpoint:

```json
{
  "total_earned": "123.456789",
  "available_balance": "45.000000",
  "accrual_counts": { "order": 0, "topup": 0, "subscription": 0, "total": 0 }
}
```

Amounts are 6dp strings, not JSON numbers. A missing source type is `0`. `available_balance` is `max(0, balance)`. Negative balance stays in the database and is visible in Django admin only.

`GET /api/v1/orgs/partner/customers` is current attributions only. Aggregates are annotated in one query. Parameters: `page`, `page_size` default 50 max 100, `sort` `attributed_at` | `total_partner_earned` | `accrual_count`, `order` `asc` | `desc`. Default `sort=total_partner_earned&order=desc`. The backend always adds `customer_id ASC`. The client does not send it. `q` is trimmed. Empty `q` is no filter. A decimal `User.id` is an exact `customer_id` inside this channel. Otherwise exact `email` or exact `display_name` inside this channel. No `icontains`, no prefix, no search outside the channel. A miss is an empty page. Zero earnings is `"0.000000"` and `accrual_count` 0.

Customer object: `customer_id`, full `email`, `display_name` (`""` when unset), `attributed_at`, `total_partner_earned`, `accrual_count`. The portal shows trimmed `display_name` when it is non-empty, otherwise `email`. Grant selects `customer_id` from that row.

`GET /api/v1/orgs/partner/grants` reads `PartnerCreditGrant` for the resolved channel. Sort `created_at` | `amount`, default `created_at desc`, tie-break `grant_id ASC`. No `q`. Fields: `grant_id`, `customer_id`, full `email` or `null`, `display_name` (`""` when unset), `amount`, `granted_by` `{user_id, email, display_name}` or `null`, `created_at`. Do not expose the user-id snapshot, ledger ids, or team account id. The same display rule applies: trimmed `display_name`, otherwise `email`.

`GET /api/v1/orgs/partner/invite-link` returns `url`, `is_active`, `created_at`, `regenerated_at` (`null` until regenerated). No separate token field. An inactive link is 200 with `is_active` false. Viewer, admin, and owner may read.

`POST …/invite-link/regenerate` has no body, owner only, not idempotent. `SELECT … FOR UPDATE` the link. The old token dies in that transaction. Logs must not contain the new URL or token. Viewer and admin get `403 partner_invite_forbidden`.

`POST …/invite-link/activate` and `…/deactivate` have no body, owner only, lock the canonical link, and do not rotate the token. Already-active activate and already-inactive deactivate return the current state and write no audit event. After deactivate, public join of that token is the generic 404. Confirmation of a pending row for that link creates no attribution while the link stays inactive. Activate turns the same token back on. These portal writes do not apply to campaign links.

The invite card shows the URL from GET. Copy is available to viewer, admin, and owner. Activate or Deactivate, and Regenerate, are owner only, with confirmation. After POST, refetch GET. Do not assume the new URL or `is_active`.

Summary and invite-link load independently. Each waits on its own skeleton. `0` is not a pre-response state and not an error substitute. One block’s network or `5xx` failure does not clear the other. No local summary math.

Give credit exists only on `/customers`, owner and admin, for that row’s `customer_id`. The modal shows the customer label (`display_name` or full email) and `available_balance` as display only. The client rejects `amount <= 0` and amount above the displayed balance. Confirm is the confirmation before POST. `idempotency_key` is minted for that attempt, reused on network retry, and minted again when the amount changes. Success closes the modal and refetches summary and grants. `409 insufficient_funds` refreshes balance. `409 customer_attribution_changed` closes the modal and reloads customers.

`/grants` is read-only. Null email or null `granted_by` displays as an em dash. Navigation for viewer, admin, and owner is Dashboard, Customers, Grants, on that team host only. No Settings, Team, Billing, Reports, Earnings, or consumer links.

Portal screens render API data as text. The customer label and the invite URL are not placed in `innerHTML`. Bearer tokens are not logged. Team routes `/`, `/customers`, and `/grants` load no third-party scripts. `/login` may load only Google GIS and Cloudflare Turnstile.

### CORS and hosts

Production allowlist contains `https://team.roamkit.net` for `https://api.roamkit.net`. Staging allowlist contains only `https://team.staging.roamkit.net` for `https://api.staging.roamkit.net`. No `CORS_ALLOW_ALL_ORIGINS`, no `*.roamkit.net`, no origin regex. `/join` does not depend on this CORS.

`team.roamkit.net` resolves through production DNS and production Traefik to the existing `roamkit-web-production` service, adding `Host(team.roamkit.net)` beside `roamkit.net` and `www.roamkit.net`. `team.staging.roamkit.net` adds `Host(team.staging.roamkit.net)` on the existing staging `roamkit-web` service beside `staging.roamkit.net`. No new container. Production must not list the staging team host, and the reverse. After deploy, `GET /` on each team host returns the partner shell. The same container with a consumer Host still returns the consumer page.

### OpenAPI

drf-spectacular and committed `openapi/openapi.yaml`, `bearerAuth`. It documents the locked bodies. It is not a second source of fields. Paths:

```text
GET  /api/v1/orgs/partner/summary
GET  /api/v1/orgs/partner/customers
GET  /api/v1/orgs/partner/grants
GET  /api/v1/orgs/partner/invite-link
POST /api/v1/orgs/partner/invite-link/regenerate
POST /api/v1/orgs/partner/invite-link/activate
POST /api/v1/orgs/partner/invite-link/deactivate
POST /api/v1/billing/partner-grants/
```

`GET /join/<token>` and the internal consume endpoint are not in this schema. CI fails if a partner response grows a property outside the locked set, including `net_price`, `margin`, `revenue_share_percent`, and `team_account_id`. The allowed `email` property is the full address. `display_name` is allowed and is `""` when the user has not set one.

`User.display_name` is a label, not an identity. Empty means callers show `email`. It does not replace login, password reset, or invite delivery. `google_name` is not a fallback.

### Rate limit

Numbers are not part of this ADR. Join is a gentle per-IP limit. Over the limit: `429`, no cookie, no pending. Grant is stricter per authenticated user and `PartnerChannel`. A new grant over the limit writes nothing. An idempotent replay of an existing grant still returns it. Portal reads are gentler than grant.

### Portal write audit

Structured log, not the grant row and not the ledger. Written after a successful commit. Transaction failure, `409`, and replay of an existing grant do not emit `partner_grant.created`. Activate or deactivate that does not change `is_active` does not emit an event.

```text
partner_invite.regenerated
partner_invite.activated
partner_invite.deactivated
partner_grant.created
```

Fields: `actor_user_id`, `partner_channel_id`, `organization_id`, `action`, `created_at`, `request_id`. Grant also has `target_customer_id` and `amount` as a 6dp decimal string. No email, Bearer token, invite URL, or full invite token.

## Concurrency and idempotency

| Case | Rule |
|------|------|
| Same commercial event twice | One accrual and one `partner_margin` ledger row. |
| Same grant key, same customer, same amount | Return the original grant. No second ledger pair. |
| Same grant key, different customer or amount | `409 idempotency_key_conflict`. |
| Transfer vs fulfillment | First commit wins. Snapshot comes from the locked attribution. |
| Channel deactivate or percent change vs fulfillment | First commit wins. Read `is_active` and percent only after the channel lock. |
| Two renewal workers | Cycle-row lock. At most one debit and one accrual per `(subscription_id, billing_date)`. |
| Two first consumes | One attribution. The other response is `noop`. |
| Consume when an attribution exists | `noop` for the same or another partner. The row does not change. |
| Two grants that together exceed balance | Amounts that commit do not exceed balance. The loser writes no ledger. This follows the team Account lock inside `CreditService`. |

Grant idempotency scope is the resolved channel plus the key.

## Operations and reconcile

Django admin for `PartnerChannel` shows, read-only, in one place: organization, `is_active`, `revenue_share_percent`, live team `Account.balance` (including negative), the canonical link’s `is_active`, count of current attributions, `SUM(partner_share)` for that channel, and the count of grants. The token and the full join URL are not shown on that screen. A missing team Account is shown as missing, not as zero. The only channel writes remain activate, deactivate, and the percent service. Assign and transfer stay on attribution admin.

`PartnerInviteLink` admin is where campaign links are created. It may change `name`, `bonus_amount`, and `is_active`. `token` stays readonly. Regenerate is the separate service, not an edit of the token field. After the first visit, `partner_channel`, `source`, `campaign`, and `content` are readonly. A bonus change is recorded in the Django admin log.

`PartnerMarginAccrual` and `PartnerCreditGrant` are audit-only, filtered by channel, customer, and date. No edit and no delete. `list_price`, `net_price`, and `margin` stay visible there and not in the portal. The renewal-cycle row is read-only. `CustomerAttributionHistory` is view-only.

`partner_channel_reconcile` checks and reports drift. It does not `INSERT`, `UPDATE`, or `DELETE`, does not call `CreditService`, and does not change `Account.balance`. It is not `cleanup_expired_partner_attributions`.

- Every accrual has exactly one `partner_margin` ledger entry, and every such ledger entry has exactly one accrual.
- Every grant has exactly one `partner_grant_out` and one `partner_grant_in`. A ledger row of those types without that grant is the same drift.
- The absolute amounts of those two rows match each other and match `PartnerCreditGrant.amount`.
- `partner_grant_out` is on that channel’s team Account. `partner_grant_in` is on the customer’s personal Account.
- `partner_share` equals the positive delta of its `partner_margin` ledger row.
- Every `PartnerChannel` has at least one `PartnerInviteLink`. A channel with none is drift. Extra campaign links are not drift.

A clean run reports no drift. Drift is reported and not repaired.

## Rollout

Schema and code stay compatible with today’s consumer flow while the flag is off. After the schema migration, existing orders, topups, subscriptions, and deposits behave as they do today. New tables do not fill themselves.

```text
1. This ADR Accepted, plus the narrow ADR 020 amendment
2. Schema migrations and the partner ledger reference types, including `partner_invite_bonus`
3. Deploy API with PARTNER_CHANNEL_ENABLED=false
4. Margin, grant, and API code and tests, flag still false
5. Web portal, CORS, and infra
6. Smoke and E2E
7. Admin creates the PartnerChannel and binds existing customers
8. Only then PARTNER_CHANNEL_ENABLED=true
```

Production step 6 still sees the flag off: the consumer flow is unchanged and the portal returns `partner_channel_disabled`. Happy-path E2E with the flag on runs in a test environment and does not flip the production flag.

PRs after Accept stay separate. Deploy follows the list above.

1. **Schema.** `Topup.net_price_usd` nullable `Decimal(20,6)` without backfill. Renewal-cycle row. Partner tables, constraints, and indexes below. Ledger types. No `partner_margin_reversal`. Reversible migrations. No backfill of old purchases or old earnings.
2. **Margin service.** Tests for the formula, idempotency, skips, flag, inactive channel, transfer versus fulfillment, and percent or `is_active` races. The refund path is not changed.
3. **Grant service.** Tests for role, foreign customer, transferred customer, negative or insufficient balance, replay, and payload conflict.
4. **API.** Portal routes, internal consume (`created` / `noop` / `ignored`), activate from `PendingPartnerAttribution`. Cookie and `GET /join/<token>` are not this PR.
5. **Web.** Host shell, team login and Google, logout and `401` cleanup, `GET /join/<token>` redirecting to `/register?from=invite`, same-origin auth proxies, `POST /join/complete/consume` for an already signed-in visitor, portal pages. Email submit does not delete the cookie. A successful Google auth that had a cookie, and a consume `created` / `noop` / `ignored`, may delete it. `401` / `5xx` on consume do not. Ordinary consumer logout does not.
6. **Infra.** DNS and Traefik only, plus the explicit CORS origins. No new service.

### Database constraints

```text
PartnerChannel
  UNIQUE organization
  CHECK 0 <= revenue_share_percent <= 100

PartnerInviteLink
  UNIQUE token
  CHECK bonus_amount >= 0
  partner_channel is not unique

InviteVisit
  invite_link PROTECT
  INDEX (invite_link, created_at)  name bill_invite_visit_link_at

CustomerAttribution
  UNIQUE user
  invite_visit PROTECT, nullable, not unique
  INDEX (partner_channel, attributed_at)

PendingPartnerAttribution
  UNIQUE user
  invite_visit PROTECT, nullable
  INDEX (expires_at)

PartnerMarginAccrual
  UNIQUE (source_type, source_id)
  UNIQUE ledger_entry
  INDEX (partner_channel, created_at)
  INDEX (partner_channel, customer_user)
  INDEX (customer_user)

PartnerCreditGrant
  UNIQUE (partner_channel, idempotency_key)
  UNIQUE debit_ledger_entry
  UNIQUE credit_ledger_entry
  INDEX (partner_channel, created_at)
  INDEX (partner_channel, customer_user)

Renewal cycle
  UNIQUE (subscription_id, billing_date)
  UNIQUE (subscription_id) WHERE status IN ('pending', 'paused', 'failed')

CustomerAttributionHistory
  INDEX (user, changed_at)
```

### Onboarding after the flag exists

Not application code and not raw SQL. Do not hardcode a user id.

1. The operator’s Organization exists and has a team Account. If not, `create_organization()`. Do not convert the personal Account.
2. Create `PartnerChannel` at `revenue_share_percent = 50`. That same step creates the canonical invite link with empty classification and `bonus_amount = 0.000000`.
3. Bind existing customers with Assign (`source=admin`). If they already have a partner, call `transfer_customer_attribution()`. That changes only `partner_channel` and writes no `CustomerAttributionHistory` row.
4. Only then set `PARTNER_CHANNEL_ENABLED=true`.

No retroactive accruals. Commission starts on a new `fulfilled` or `renewed` after attribution, an active channel, and the flag. An optional `admin_adjust` on the team Account may happen before the first earning. The operator’s own pricing profile stays on the personal Account. Those personal purchases do not earn partner share.

## Non-goals

- Amending [ADR 019](./019-account-pricing-profiles.md), including `discount_percent` or `money_round()`.
- Amending [ADR 010](./010-polygon-usdt-prepaid-credits.md) money invariants.
- Turning Organization into a money owner, or a white-label store.
- Changing `OrganizationInvite` or the ADR 020 membership permission matrix.
- Cash-out, team-to-team transfer, granting an arbitrary user, or spending the team wallet as the customer’s payment method.
- Automatic reversal of partner margin on refund. Refunds stay on the existing support path.
- A public accrual list or a QR code.
- Hard delete of a channel, invite link, invite visit, accrual, grant, history row, or renewal cycle.

Not part of this feature:

- A partner-portal list of campaign links.
- An analytics or report UI.
- A UTM builder.
- A QR generator.
- Link-level expiration.
- Click limits.
- Location or device fingerprinting.
- Inferring `source` from Referer.
- Self-service bonus edit.
- Self-service attribution transfer.
- A repair mode for `partner_channel_reconcile`.
- Rate-limit numbers, proxy log redaction, and a later anonymization product.
- Implementing schema or API in the Accept step of this ADR.

## Consequences

### Positive

- Partner commission is a new credit source on the existing team Account, not a second ledger and not a customer discount.
- Attribution, accrual, and grant stay auditable after transfer and after user deletion.
- The host-only cookie and the server-verified pending payload keep the invite token out of the browser and stop Next.js from asserting a partner id.
- The flag keeps production behavior unchanged until ops binds real customers.

### Negative / trade-offs

- `team.roamkit.net` is a second origin with its own Bearer storage and Google JavaScript origin.
- Renewal needs a durable per-cycle price row because `price_per_period` is not a snapshot.
- Team balance and `SUM(partner_share)` are allowed to diverge. Ops reads the difference in Django admin, not in the portal.
- ADR 020’s “B2B reseller portal” non-goal is no longer absolute. The allowed surface is only this channel.

### Binding after Accept

Do not add a partner discount, a public accrual API, a channel delete, or an automatic margin reversal without a new ADR. Do not store a team Account FK on `PartnerChannel`. Do not let the browser or Next.js choose `partner_channel_id`.

## ADR 012 compliance

| Rule | Status |
|------|--------|
| CreditService only | ✅ Margin credit, both grant legs, and the registration bonus call `CreditService` only. |
| Ledger SoT | ✅ Balance stays a cache. The portal does not recompute it. |
| Append-only ledger | ✅ No update or delete of ledger rows. No `partner_margin_reversal` in v1. |
| Account owner | ✅ Credits and debits target `billing.Account`. `PartnerChannel` has no Account FK. |
| Idempotent | ✅ `partner-margin:{source_type}:{source_id}`, unique `(partner_channel, idempotency_key)`, and `invite-registration-bonus:<user_id>`. |
| Single DB transaction + lock | ✅ Fulfillment and grant lock orders are normative above. The registration bonus is in the same transaction as the attribution insert. |
| Snapshot event | ✅ The accrual and renewal-cycle rows are the price snapshots. v1 adds no new bus event; the locked structured logs are the operational record. |
| `/api/v1/billing/` only | ✅ The money write is `POST /api/v1/billing/partner-grants/`. Portal reads live under `/api/v1/orgs/partner/`, the ADR 020 org family, and are not a second money API. |
| Feature flag / gate | ✅ `PARTNER_CHANNEL_ENABLED` default off. |
| Architecture tests | ✅ Required in the margin, grant, and API PRs: wholesale fields stay out of partner responses; OpenAPI property set; backend matrix in this ADR. |
| Does not revise ADR 010 invariants | ✅ |
| Decimal(20,6) | ✅ Accrual money fields, grant amount, `bonus_amount`, and `bonus_amount_snapshot`. `revenue_share_percent` is `Decimal(5,2)` by the CHECK above. |

## Related

- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — money path; gains `partner_invite_bonus` only. Invariants unchanged.
- [ADR 012](./012-billing-extensibility-rules.md) — credit-source constitution
- [ADR 019](./019-account-pricing-profiles.md) — customer pricing; unchanged
- [ADR 020](./020-organization-team-accounts.md) — Organization and team Account; non-goal narrowed
- [ADR index](../../ADR_INDEX.md)
