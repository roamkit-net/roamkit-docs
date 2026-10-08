# ADR 023: Partner Channel

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-10 |
| Amended | 2026-10-08 (user-id snapshots use the same scalar type as `User.id`; `source_id` uses each model's canonical primary key) |
| Deciders | Solo operator (architecture lock before schema / API) |
| Relates to | [ADR 010](./010-polygon-usdt-prepaid-credits.md), [ADR 012](./012-billing-extensibility-rules.md), [ADR 019](./019-account-pricing-profiles.md) (unchanged), [ADR 020](./020-organization-team-accounts.md) (narrow amendment only) |

## Context

RoamKit can attach a permanent invite link to one Organization so that new customers who arrive through that link are attributed to that partner. Attributed customers stay on their personal Accounts and pay through the existing purchase path. A share of RoamKit’s margin is credited to the Organization’s existing team Account. The partner sees those customers and that balance in a portal on `team.roamkit.net`, and may move credits from the team Account to a currently attributed customer, only up to the team balance.

[ADR 020](./020-organization-team-accounts.md) already has Organization, Membership, and exactly one team `billing.Account`. Its invite creates Membership, not a customer relationship. Its non-goal “B2B reseller portal” is narrowed by the amendment in that ADR: this document is that narrow channel, not a white-label store. Organization remains not the owner of money.

[ADR 019](./019-account-pricing-profiles.md) is unchanged. `discount_percent` is still the share of margin given to the buying account. It is not a partner commission.

No user id is hardcoded. Binding a specific operator and that operator’s existing customers is ops after the flag exists.

## Decision

Add a **Partner Channel** credit source that satisfies [ADR 012](./012-billing-extensibility-rules.md) and does not revise [ADR 010](./010-polygon-usdt-prepaid-credits.md) or [ADR 019](./019-account-pricing-profiles.md).

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

`LedgerReferenceType` stays a `CharField` (`max_length` 32). Reverse of that migration must not delete ledger rows. If reverse would narrow allowed values, it stops when any row uses one of these three types.

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
| 10 | Public partner responses do not include `net_price`, `margin`, `revenue_share_percent`, `team_account_id`, or a full email. |

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

Token is URL-safe, not sequential, does not encode `organization_id`, at least 128 bits (`secrets.token_urlsafe(24)` or equivalent). `UNIQUE(token)` collision retries a new token inside the same transaction. A channel without a link does not commit. After commit every channel has exactly one link.

The channel is never physically deleted, even with no children. No admin delete, no delete service, the model refuses delete. Shutdown is `is_active=False` only. An Organization that has a channel cannot be deleted. Formal archival is out of v1.

`is_active` and `revenue_share_percent` are not saved by a raw or inline edit. Both go through a service that `SELECT … FOR UPDATE` locks that `PartnerChannel` row only, the same row the margin path locks. It does not lock attribution or the team Account.

`is_active=False` stops new accruals only. Existing team balance may still be granted. The portal still does not see `net_price` or `margin`.

### Feature flag off

`PARTNER_CHANNEL_ENABLED=false` creates no new pending row, no new attribution, no new accrual, and no team credit. Portal and grant return `404` with `{"code":"partner_channel_disabled"}`. Public `GET /join/<token>` stays a generic 404 with no JSON and no cookie. Fulfillment still succeeds. The margin path logs info `partner_margin.flag_disabled`. Existing rows and balances are unchanged. Django admin still shows them.

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

`CustomerAttribution`: `user` (unique, PROTECT), `partner_channel`, `source` (`invite_link` | `admin`), `invite_link` (nullable), `invite_token` snapshot (nullable), `attributed_at`, `created_at`.

A link does not change an existing row. Channel deactivation does not delete it. Reactivation resumes commission for the same partner. The first admin bind is `source=admin` and writes no history row.

Django admin has no form that saves `CustomerAttribution` or edits `partner_channel`. Two service actions only:

| Action | Rule |
|--------|------|
| Assign customer to partner | User + channel, `source=admin`, actor required. Rejected if an attribution exists. No history row. Actor stays in the Django admin log. |
| Transfer customer to another partner | New channel and a required reason. Only `transfer_customer_attribution()`. Rejected if no attribution exists. |

Transfer locks the attribution row, writes `CustomerAttributionHistory` (`from_partner_channel`, `to_partner_channel`, `changed_by`, `change_reason`, `changed_at`, previous `attributed_at`), then retargets. It does not move old accruals or team balance. Later purchases follow the new partner. History index is `(user, changed_at)`. Do not add `created_at` or `customer_user_snapshot` for that index.

### Pending row

`PendingPartnerAttribution`: OneToOne `user`, `partner_channel`, `invite_token_snapshot`, `expires_at`, `created_at`. Index `(expires_at)`. Only for the inactive user created by register. One pending per user.

`expires_at` decides validity. `activate` checks `expires_at > now()` first and deletes only that expired row, even if the periodic job has not run. Expired, regenerated, or inactive link: activate creates no attribution and deletes that pending row. Successful attribution deletes it. If the user already has an attribution, the pending row is discarded. Auth does not bulk-clean other rows.

`cleanup_expired_partner_attributions` deletes only `PendingPartnerAttribution` where `expires_at <= now()`. It does not touch attribution, history, accruals, or ledger. A late job does not change activate’s outcome.

The link does not expire. Pending lasts at most 24 hours and also ends when the link is regenerated or deactivated. A user who opens a link today and registers two months later is not attributed from it.

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

## Invite and auth flow

One permanent reusable `PartnerInviteLink` per channel. Not campaign links, not email-bound, not single-use. No `expires_at`. No QR in v1.

Fields: `partner_channel` (unique), `token` (unique), `is_active`, `created_at`, `regenerated_at` (nullable). The link cannot be deleted. Token changes only via regenerate. `is_active` changes only via activate or deactivate. Channel shutdown does not flip `link.is_active`.

Public URL: `https://roamkit.net/join/<token>`. `utm_*` is ignored and does not create a link.

`GET /join/<token>` for an unknown, inactive, or regenerated-away token returns the same generic 404. No cookie, no attribution, no ledger, no channel state. Logs must not contain the full token or join URL. Optional fingerprint: `sha256(token)[0:12]`. Proxy access logs may still see the path. Redacting them is out of v1. The route does not embed the token in HTML or JS. It validates the token and stores a signed cookie. `Cache-Control: no-store`.

Combinations:

| Channel | Link | Result |
|---------|------|--------|
| Off | On | New customers can still bind. Purchases earn nothing until the channel is active. |
| On | Off | No new link attributions. Existing attributed customers still earn. |

### Cookie

`partner_pending` is host-only on the consumer host (`roamkit.net`, and the same on `staging.roamkit.net`): `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, `Max-Age=86400`, no `Domain=.roamkit.net`. `api.roamkit.net` and `team.roamkit.net` do not receive it. The value is a server-signed payload, not the raw token. JS cannot read it.

It is join context until consume. It is not part of general login or logout. An ordinary consumer logout does not delete it. Login, register, and Google do not delete it. The browser drops it at `Max-Age`. Before auth, the last valid `/join/<token>` replaces it. An invalid join does not clear an existing cookie. After a server pending row or a `CustomerAttribution` exists, a later link does not overwrite that context.

### `/join/complete`

`GET /join/<token>` cannot see the Bearer in `localStorage` and does not attribute, even if the user is logged in. A valid link sets the cookie and redirects to `/join/complete` on the consumer host. No Bearer → `/login?next=/join/complete`. After email or Google, return there. Login and Google do not themselves attribute.

If a Bearer exists, the browser calls only:

```text
POST https://roamkit.net/join/complete/consume
Authorization: Bearer <user-token>
```

The same route exists on `staging.roamkit.net`. It is not on the team host. The browser does not send `partner_channel_id`, the invite token, or an organization id, and never sees the cookie body.

Next.js is not the authority for user id or partner id. It forwards the same Bearer and the signed pending payload. The signature must be verifiable by the API. A local Next.js check is not enough. The API does not accept a plain `user_id` or `partner_channel_id` as proof. This endpoint is not a public partner route and is not in the public partner OpenAPI schema.

The API sets `request.user` from the Bearer. From the signed payload it reads the channel, token snapshot, and `expires_at`, and verifies the signature itself.

If that user already has a `CustomerAttribution`, stop. Return `{"status":"noop"}` with 200, whether the pending partner is the same or another. Do not change the row. Two parallel first consumes: one inserts, the other sees the row and returns `noop`, not an integrity error.

Only when no attribution exists does the API, under lock, re-check the flag, `PartnerInviteLink.is_active`, snapshot token equals the current token, pending not expired, and still no attribution. Then it creates the row and returns `{"status":"created"}`. Otherwise `{"status":"ignored"}` with 200. `ignored` covers an invalid, expired, or old pending, an inactive link, and the flag off, and does not say which. A system error stays 5xx. An invalid Bearer stays 401 and is not one of the three statuses.

```text
created / noop / ignored → Next.js deletes partner_pending → consumer page
401 / 5xx                → cookie stays; 5xx shows a local retry
```

The body on 200 is exactly `{"status":"created"|"noop"|"ignored"}`. No partner id, token, organization id, or reason.

Register of an inactive user still writes `PendingPartnerAttribution`. `activate` still reads that row, not the cookie. If the cookie later names partner B, activate still uses partner A from the pending row, and only while that snapshot still matches the current active link.

### Team host auth

`localStorage` is per origin. The token on `roamkit.net` is not the token on `team.roamkit.net`. No copy and no `postMessage`.

Login starts and ends on the same team host. Google stays GIS `ux_mode: "popup"`. The callback is the in-page callback, then `POST /api/v1/auth/google/`. No redirect mode and no OAuth redirect to `roamkit.net`. Authorized JavaScript origins gain exactly `https://team.roamkit.net` and `https://team.staging.roamkit.net`. No wildcard.

`next` on the team host may be only `/`, `/customers`, or `/grants`. Anything else, including `/me/esims` and absolute URLs, falls back to `/`.

Logout on `team.roamkit.net` deletes only that origin’s Bearer and returns to `/login` on that host. Staging deletes only the staging team token. Neither touches consumer `localStorage` nor `partner_pending`. Consumer logout does not touch the team token.

`401` from a portal call (`summary`, `customers`, `grants`, invite-link, regenerate, activate, deactivate, `partner-grants`) deletes that team Bearer and goes to `/login?next=<current allowed route>`. A `401` from the login form itself stays on the form.

## Portal and API

The shell is the existing Next.js app. Middleware reads `Host` before HTML.

```text
roamkit.net | www.roamkit.net | staging.roamkit.net → consumer web
team.roamkit.net | team.staging.roamkit.net         → partner shell
```

No public `roamkit.net/team` and no query flag. v1 team routes are `/`, `/customers`, and `/grants`. `/` is the dashboard: `total_earned`, `available_balance`, `accrual_counts`, and one invite card. Unauthenticated users go to `/login` on that same host.

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
400 invalid_amount
400 invalid_page | invalid_page_size | invalid_sort | invalid_order
400 invalid_query          customers only, q longer than 254 after trim
```

`customer_not_found` must not reveal that the user id exists at another partner. `customer_attribution_changed` means the customer was transferred off this channel under the attribution lock. A valid page past the end is not an error: return `count`, the requested `page`, `page_size`, and `results: []`.

`401` on the team host is the auth cleanup above, not a portal error screen. `403`, `404 partner_channel_disabled`, and `409 partner_context_ambiguous` replace the page with the portal screen (access denied, feature unavailable, or contact support with no org list). Network and `5xx` stay on that block with retry.

### Read contracts

`Cache-Control: no-store` on the whole Partner Channel HTTP surface, including errors and `POST /join/complete/consume`.

`GET /api/v1/orgs/partner/summary` is the only dashboard numbers endpoint:

```json
{
  "total_earned": "123.456789",
  "available_balance": "45.000000",
  "accrual_counts": { "order": 0, "topup": 0, "subscription": 0, "total": 0 }
}
```

Amounts are 6dp strings, not JSON numbers. A missing source type is `0`. `available_balance` is `max(0, balance)`. Negative balance stays in the database and is visible in Django admin only.

`GET /api/v1/orgs/partner/customers` is current attributions only. Aggregates are annotated in one query. Parameters: `page`, `page_size` default 50 max 100, `sort` `attributed_at` | `total_partner_earned` | `accrual_count`, `order` `asc` | `desc`. Default `sort=total_partner_earned&order=desc`. The backend always adds `customer_id ASC`. The client does not send it. `q` is trimmed. Empty `q` is no filter. A decimal `User.id` is an exact `customer_id` inside this channel. Otherwise `email__iexact` inside this channel. No `icontains`, no prefix, no search outside the channel. A miss is an empty page. The full email is never returned. Zero earnings is `"0.000000"` and `accrual_count` 0.

Customer object: `customer_id`, masked email (first local character + `***` + domain), `attributed_at`, `total_partner_earned`, `accrual_count`. No `display_name`. Full email stays in Django admin. Grant selects `customer_id` from that row.

`GET /api/v1/orgs/partner/grants` reads `PartnerCreditGrant` for the resolved channel. Sort `created_at` | `amount`, default `created_at desc`, tie-break `grant_id ASC`. No `q`. Fields: `grant_id`, `customer_id`, masked `email` or `null`, `amount`, `granted_by` `{user_id, email}` or `null`, `created_at`. Do not expose the user-id snapshot, ledger ids, or team account id.

`GET /api/v1/orgs/partner/invite-link` returns `url`, `is_active`, `created_at`, `regenerated_at` (`null` until regenerated). No separate token field. An inactive link is 200 with `is_active` false. Viewer, admin, and owner may read.

`POST …/invite-link/regenerate` has no body, owner only, not idempotent. `SELECT … FOR UPDATE` the link. The old token dies in that transaction. Logs must not contain the new URL or token. Viewer and admin get `403 partner_invite_forbidden`.

`POST …/invite-link/activate` and `…/deactivate` have no body, owner only, lock the link, do not rotate the token. Already-active activate and already-inactive deactivate return the current state and write no audit event. After deactivate, public join is the generic 404. Pending with that token cannot be consumed until activate turns the same token back on.

The invite card shows the URL from GET. Copy is available to viewer, admin, and owner. Activate or Deactivate, and Regenerate, are owner only, with confirmation. After POST, refetch GET. Do not assume the new URL or `is_active`.

Summary and invite-link load independently. Each waits on its own skeleton. `0` is not a pre-response state and not an error substitute. One block’s network or `5xx` failure does not clear the other. No local summary math.

Give credit exists only on `/customers`, owner and admin, for that row’s `customer_id`. The modal shows masked email and `available_balance` as display only. The client rejects `amount <= 0` and amount above the displayed balance. Confirm is the confirmation before POST. `idempotency_key` is minted for that attempt, reused on network retry, and minted again when the amount changes. Success closes the modal and refetches summary and grants. `409 insufficient_funds` refreshes balance. `409 customer_attribution_changed` closes the modal and reloads customers.

`/grants` is read-only. Null email or null `granted_by` displays as an em dash. Navigation for viewer, admin, and owner is Dashboard, Customers, Grants, on that team host only. No Settings, Team, Billing, Reports, Earnings, or consumer links.

Portal screens render API data as text. Masked email and the invite URL are not placed in `innerHTML`. Bearer tokens are not logged. Team routes `/`, `/customers`, and `/grants` load no third-party scripts. `/login` may load only Google GIS and Cloudflare Turnstile.

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

`GET /join/<token>` and the internal consume endpoint are not in this schema. CI fails if a partner response grows a property outside the locked set, including `net_price`, `margin`, `revenue_share_percent`, `team_account_id`, and a full email. The allowed `email` property is the mask.

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

Django admin for `PartnerChannel` shows, read-only, in one place: organization, `is_active`, `revenue_share_percent`, live team `Account.balance` (including negative), invite-link `is_active`, count of current attributions, `SUM(partner_share)` for that channel, and the count of grants. The token and the full join URL are not shown. A missing team Account is shown as missing, not as zero. The only channel writes remain activate, deactivate, and the percent service. Assign and transfer stay on attribution admin.

`PartnerMarginAccrual` and `PartnerCreditGrant` are audit-only, filtered by channel, customer, and date. No edit and no delete. `list_price`, `net_price`, and `margin` stay visible there and not in the portal. The renewal-cycle row is read-only. `CustomerAttributionHistory` is view-only.

`partner_channel_reconcile` checks and reports drift. It does not `INSERT`, `UPDATE`, or `DELETE`, does not call `CreditService`, and does not change `Account.balance`. It is not `cleanup_expired_partner_attributions`.

- Every accrual has exactly one `partner_margin` ledger entry, and every such ledger entry has exactly one accrual.
- Every grant has exactly one `partner_grant_out` and one `partner_grant_in`. A ledger row of those types without that grant is the same drift.
- The absolute amounts of those two rows match each other and match `PartnerCreditGrant.amount`.
- `partner_grant_out` is on that channel’s team Account. `partner_grant_in` is on the customer’s personal Account.
- `partner_share` equals the positive delta of its `partner_margin` ledger row.
- Every `PartnerChannel` has exactly one `PartnerInviteLink`.

A clean run reports no drift. Drift is reported and not repaired.

## Rollout

Schema and code stay compatible with today’s consumer flow while the flag is off. After the schema migration, existing orders, topups, subscriptions, and deposits behave as they do today. New tables do not fill themselves.

```text
1. This ADR Accepted, plus the narrow ADR 020 amendment
2. Schema migrations and the three ledger reference types
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
5. **Web.** Host shell, team login and Google, logout and `401` cleanup, `GET /join/<token>`, `POST /join/complete/consume`, portal pages. `created` / `noop` / `ignored` delete the cookie. `401` / `5xx` do not. Ordinary consumer logout does not.
6. **Infra.** DNS and Traefik only, plus the explicit CORS origins. No new service.

### Database constraints

```text
PartnerChannel
  UNIQUE organization
  CHECK 0 <= revenue_share_percent <= 100

PartnerInviteLink
  UNIQUE partner_channel
  UNIQUE token

CustomerAttribution
  UNIQUE user
  INDEX (partner_channel, attributed_at)

PendingPartnerAttribution
  UNIQUE user
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
2. Create `PartnerChannel` at `revenue_share_percent = 50`. That same step creates the invite link.
3. Bind existing customers with Assign (`source=admin`). If they already have a partner, Transfer with history.
4. Only then set `PARTNER_CHANNEL_ENABLED=true`.

No retroactive accruals. Commission starts on a new `fulfilled` or `renewed` after attribution, an active channel, and the flag. An optional `admin_adjust` on the team Account may happen before the first earning. The operator’s own pricing profile stays on the personal Account. Those personal purchases do not earn partner share.

## Non-goals

- Amending [ADR 019](./019-account-pricing-profiles.md), including `discount_percent` or `money_round()`.
- Amending [ADR 010](./010-polygon-usdt-prepaid-credits.md) money invariants.
- Turning Organization into a money owner, or a white-label store.
- Changing `OrganizationInvite` or the ADR 020 membership permission matrix.
- Cash-out, team-to-team transfer, granting an arbitrary user, or spending the team wallet as the customer’s payment method.
- Automatic reversal of partner margin on refund. Refunds stay on the existing support path.
- A public accrual list, campaign links, or QR.
- Hard delete of a channel, invite link, accrual, grant, history row, or renewal cycle.
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
| CreditService only | ✅ Margin credit and both grant legs call `CreditService` only. |
| Ledger SoT | ✅ Balance stays a cache. The portal does not recompute it. |
| Append-only ledger | ✅ No update or delete of ledger rows. No `partner_margin_reversal` in v1. |
| Account owner | ✅ Credits and debits target `billing.Account`. `PartnerChannel` has no Account FK. |
| Idempotent | ✅ `partner-margin:{source_type}:{source_id}` and unique `(partner_channel, idempotency_key)`. |
| Single DB transaction + lock | ✅ Fulfillment and grant lock orders are normative above. |
| Snapshot event | ✅ The accrual and renewal-cycle rows are the price snapshots. v1 adds no new bus event; the locked structured logs are the operational record. |
| `/api/v1/billing/` only | ✅ The money write is `POST /api/v1/billing/partner-grants/`. Portal reads live under `/api/v1/orgs/partner/`, the ADR 020 org family, and are not a second money API. |
| Feature flag / gate | ✅ `PARTNER_CHANNEL_ENABLED` default off. |
| Architecture tests | ✅ Required in the margin, grant, and API PRs: wholesale fields stay out of partner responses; OpenAPI property set; backend matrix in this ADR. |
| Does not revise ADR 010 invariants | ✅ |
| Decimal(20,6) | ✅ Accrual money fields and grant amount. `revenue_share_percent` is `Decimal(5,2)` by the CHECK above. |

## Related

- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — money path; unchanged
- [ADR 012](./012-billing-extensibility-rules.md) — credit-source constitution
- [ADR 019](./019-account-pricing-profiles.md) — customer pricing; unchanged
- [ADR 020](./020-organization-team-accounts.md) — Organization and team Account; non-goal narrowed
- [ADR index](../../ADR_INDEX.md)
