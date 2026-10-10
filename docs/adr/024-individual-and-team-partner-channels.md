# ADR 024: Individual and Team Partner Channels

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-10-09 |
| Deciders | Solo operator (architecture lock before schema / API) |
| Relates to | [ADR 010](./010-polygon-usdt-prepaid-credits.md) (unchanged), [ADR 012](./012-billing-extensibility-rules.md), [ADR 019](./019-account-pricing-profiles.md) (unchanged), [ADR 020](./020-organization-team-accounts.md), [ADR 023](./023-partner-channel.md) |
| Supersedes in part | The listed parts of [ADR 023](./023-partner-channel.md) only. ADR 023 remains the historical decision for the TEAM channel that is already built. |

## Context

[ADR 023](./023-partner-channel.md) locks one partner shape: `PartnerChannel` is 1:1 with `Organization`, portal access comes only from Membership, and margin and grants settle on that Organization's team Account. That shape stays valid for a team seller.

A person who is already a customer can also become a seller without a second wallet. Their commission should land on the Personal Account they already use to buy eSIMs. Turning that Personal Account into an Organization Account would move existing purchases, balance, and eSIM ownership and would break the `personal` / `organization` identity rules in [ADR 020](./020-organization-team-accounts.md).

[ADR 010](./010-polygon-usdt-prepaid-credits.md) already makes `billing.Account` the financial owner and keeps the ledger append-only. Partnership is a business role above that Account. It is not a new Account kind.

This ADR does not authorize schema, migrations, API, frontend, admin behavior changes, or production data changes. Those require later PRs that follow this text.

## Decision

`PartnerChannel.kind` is `INDIVIDUAL` or `TEAM`. The channel is not a money owner and has no `account` FK. One server-side function, conceptually `resolve_partner_settlement_account(channel) -> billing.Account`, is the only place that maps a channel to the Account that receives commission and funds grants.

```text
PartnerChannel
├── INDIVIDUAL
│   └── owner_user
│       └── existing Personal Account
└── TEAM
    └── organization
        └── existing Organization Account
```

```text
INDIVIDUAL commission or grant source
    → Personal Account
    → CreditService
    → CreditLedgerEntry

TEAM commission or grant source
    → Organization Account
    → CreditService
    → CreditLedgerEntry
```

Where this ADR and ADR 023 disagree, this ADR wins. Everything in ADR 023 that is not listed under "What this ADR changes in ADR 023" stays in force, including invite visits, registration bonus, margin formula, accrual immutability, and the TEAM role matrix.

### What this ADR changes in ADR 023

| ADR 023 rule | Replacement |
|--------------|-------------|
| `PartnerChannel` belongs only to an Organization | `kind` plus exactly one matching owner relation |
| Portal access comes only from Membership | INDIVIDUAL ownership or TEAM Membership, rechecked every request |
| Exactly one channel and no picker | A user may own one INDIVIDUAL channel and access many TEAM channels, with explicit context selection |
| Settlement is always `organization.account` | `resolve_partner_settlement_account` |
| Grant source is always the team Account | Grant source is the resolved settlement Account |
| Self-attribution is unspecified | A person cannot economically self-refer through a channel they control or participate in |

## Ownership

| Kind | Required owner | Forbidden owner | Cardinality |
|------|----------------|-----------------|-------------|
| `INDIVIDUAL` | exactly one `owner_user` | `organization` is NULL | a User owns at most one INDIVIDUAL channel |
| `TEAM` | exactly one `organization` | `owner_user` is NULL | an Organization has at most one TEAM channel |

The future schema enforces both the XOR shape and the cardinality with database constraints. A CHECK proves the shape of one row. Uniqueness proves one INDIVIDUAL channel per user and one TEAM channel per organization.

`kind`, `owner_user`, and `organization` are immutable after insert. No service, API, or admin action may change them. After creation, admin shows those three fields read-only. Disabling or archiving an INDIVIDUAL channel leaves it owned by the same user. It is never reassigned.

Creating an INDIVIDUAL channel does not create an Account and does not change `Account.kind`. A TEAM channel continues to use the Organization Account that [ADR 020](./020-organization-team-accounts.md) already created. Personal stays personal. Organization stays organization.

## Settlement Account

```text
INDIVIDUAL
    channel.owner_user
        → that user's existing Personal Account

TEAM
    channel.organization
        → that organization's existing Organization Account
```

Margin, grant, reconcile, admin balance display, and reporting call the resolver and then use normal Account and `CreditService` rules. They do not each branch on `kind`. A missing Personal Account for an INDIVIDUAL owner, or a missing Organization Account for a TEAM channel, is a system error. It is not `insufficient_funds` and it does not create an Account as a side effect of commission or grant.

## Active partner context

A user may at the same time:

- own one INDIVIDUAL channel, and
- access one or more TEAM channels through active Organization Membership.

```text
User
├── INDIVIDUAL channel
├── TEAM channel A
└── TEAM channel B
```

The context list returns only channels that user may access, and only:

```text
channel_id
kind
label
effective role
```

An INDIVIDUAL channel's effective role is `owner`. A TEAM channel uses the active Membership role. The list does not include balance, revenue share, account ids, or tokens.

Persisting the last selected channel in browser storage or session storage is UX state only. It grants nothing.

Every partner API request that reads or changes one channel names that `PartnerChannel` in the request. The transport (path, header, or parameter) is deferred to the API PR. The server re-resolves access and role for that channel on every request. These are not authority by themselves:

- localStorage or session selection
- a client-supplied channel id that the server has not rechecked
- a previously valid selection whose membership or ownership has ended

If the named channel is missing or no longer authorized, the operation is rejected. The server does not fall back to another channel. The client refreshes the context list and the user selects again. A mutation that silently moved to another channel could spend or grant from the wrong seller.

Different browser tabs may use different contexts. That is safe only because each financial or channel-specific request carries its own channel identity and the server authorizes that identity again.

Access to a channel is not permission to mutate it. TEAM operations keep the ADR 023 matrix: `viewer` stays read-only where ADR 023 says read-only, `admin` may grant, and `owner` may also change the canonical invite link. INDIVIDUAL effective `owner` does not loosen TEAM checks.

One authorized context does not need a picker. More than one does. Zero authorized contexts stay `403 partner_access_denied`. `409 partner_context_ambiguous` is no longer the normal multi-channel case.

## Attribution

INDIVIDUAL and TEAM channels share one attribution system. There is no second attribution table and no per-kind namespace.

A customer has at most one current `CustomerAttribution`, regardless of channel kind. This is invalid:

```text
customer
├── current attribution → INDIVIDUAL A
└── current attribution → TEAM B
```

Existing link, token, and attribution identity stay as ADR 023 defined them. Backfilling existing channels to `TEAM` does not reattribute anyone.

### INDIVIDUAL self-dealing

For an INDIVIDUAL channel:

- the attributed `customer_user` is not `owner_user`
- the owner cannot be attributed to their own channel
- the owner's own order, top-up, or subscription never consumes that owner's own INDIVIDUAL link, code, or attribution
- the owner earns no partner margin from their own purchase
- the owner cannot grant from their Personal Account back onto that same Personal Account
- the grant source Account and destination Account are never the same Account

The same user may be a customer of a different PartnerChannel.

### TEAM self-dealing

An active member of the Organization that owns a TEAM channel cannot be a current customer of that same channel. This applies to every TEAM role: `owner`, `admin`, and `viewer`.

```text
INDIVIDUAL    owner_user cannot be a customer of that same channel
TEAM          active Organization member cannot be a customer of that same channel
```

For a TEAM channel:

- a user with an active `OrganizationMembership` in the owning Organization cannot be attributed to that channel
- that user's own Personal Account purchases must not earn margin for that channel
- the user may be attributed to a different PartnerChannel
- membership in Organization A does not block attribution to PartnerChannel B

Both directions fail closed:

1. Attribution is rejected when the customer is already an active member of the owning Organization.
2. Activating or inviting a Membership is rejected when that user is currently attributed to the Organization's TEAM channel.

Case 2 does not remove, transfer, or rewrite the existing attribution. The operation fails with an explicit conflict, and the attribution must be resolved first by a separate action. There is no automatic re-attribution, automatic attribution deletion, balance movement, or ledger change.

A suspended or revoked membership is not an active membership, so it is not this conflict. A pair that already violates the rule is not repaired by attribution, membership, or the TEAM backfill.

## Money

ADR 010 is unchanged. The ledger is the source of truth. `Account.balance` is a cache. Only `CreditService` mutates money. Ledger rows are append-only. Money writes stay in one database transaction, idempotent, and locked.

Commission credits the resolved settlement Account. INDIVIDUAL commission credits the existing Personal Account. TEAM commission credits the existing Organization Account. The formula, skip reasons, snapshots, and idempotency key from ADR 023 stay. Logs and reconcile use `partner_channel_id` and `settlement_account_id`. They include `organization_id` only for TEAM and `owner_user_id` only for INDIVIDUAL identity, never as a financial owner.

`total_earned` is the sum of stored `partner_share` accruals for that channel. It is a report. It is not a second spendable balance.

### INDIVIDUAL grants

```text
source       owner's Personal Account
destination  Personal Account of a customer currently attributed to that same channel
```

The grant may use the full available source balance. Deposits, admin credits, commission, refunds, and other credits are fungible in `Account.balance`. The grant does not trace which credit funded it.

Every grant, INDIVIDUAL or TEAM, keeps these rules:

- `amount > 0`
- source Account is not the destination Account
- the customer is currently attributed to that same channel
- sufficient source balance is checked while the source Account is locked
- debit and credit commit in one database transaction through `CreditService`
- Account locks have one deterministic order
- the business operation is idempotent on the existing channel-scoped key
- the ledger entries and the grant row are append-only
- nothing writes `Account.balance` directly

A personal purchase, an INDIVIDUAL grant, and any other debit of that Personal Account serialize on that Account lock.

TEAM grant direction stays Organization Account to the attributed customer's Personal Account. TEAM role checks stay as ADR 023 wrote them.

## Pricing and revenue share

`PricingProfile` and `revenue_share_percent` stay independent.

Becoming an INDIVIDUAL partner does not change `Account.kind`, assign a PricingProfile, derive revenue share from a PricingProfile, or merge the two. A Personal Account may keep its normal purchase profile. The INDIVIDUAL channel has its own `revenue_share_percent`. Activating or deactivating the channel does not change that profile. An administrator who changes pricing does that as a separate operation.

The same split holds for TEAM: Organization Account pricing does not set or follow channel revenue share.

## Migration of existing channels

This section is the future migration contract. This ADR does not add the migration.

Existing `PartnerChannel` rows become:

```text
kind           = TEAM
organization   = unchanged
owner_user     = NULL
```

Unchanged by that migration:

- channel primary keys
- organization relation
- invite links and tokens
- customer attributions
- accruals and grants
- revenue share and active flags
- Account kinds and balances
- ledger history

No money moves.

Order the future migration as:

1. Add nullable `kind` and `owner_user`.
2. Backfill every existing channel as `TEAM`.
3. Verify the invariants below.
4. Add the XOR CHECK and the uniqueness constraints.

## Lifecycle

INDIVIDUAL to TEAM is not a conversion. TEAM to INDIVIDUAL is not a conversion. `Account.kind` never flips.

If an individual seller later wants a team:

```text
User
├── existing Personal Account
├── existing INDIVIDUAL PartnerChannel   (stays, active or archived)
└── new Organization
    ├── new Organization Account
    └── new TEAM PartnerChannel
```

Nothing automatically moves Personal Account balance, eSIM ownership, or ledger history. Moving customers from the INDIVIDUAL channel to the TEAM channel is not part of this decision.

## Invariants

1. `PartnerChannel.kind` is `INDIVIDUAL` or `TEAM`.
2. Exactly one matching owner relation exists.
3. `kind`, `owner_user`, and `organization` are immutable after creation.
4. A user owns at most one INDIVIDUAL channel.
5. An organization has at most one TEAM channel.
6. A user may own an INDIVIDUAL channel and access multiple TEAM channels at the same time.
7. A client-selected channel id is never authorization.
8. Every channel-specific request revalidates access and role.
9. `PartnerChannel` has no financial Account FK.
10. The settlement Account is resolved from the owner relation.
11. All money still flows through `CreditService`.
12. INDIVIDUAL commission credits the existing Personal Account.
13. INDIVIDUAL grants may use the full available Personal Account balance.
14. A customer has at most one current attribution across both kinds.
15. A person cannot economically self-refer through a PartnerChannel they control or participate in. An INDIVIDUAL owner cannot be a customer of that channel or earn margin or a grant on their own Personal Account from it. An active member of the owning Organization cannot be a customer of that TEAM channel or earn margin for it from their own purchases. Neither attribution nor membership activation silently removes the other.
16. `PricingProfile` and `revenue_share_percent` are independent.
17. The existing-channel migration moves no money and changes no `Account.kind`.
18. INDIVIDUAL and TEAM are not an Account or channel-ownership conversion of each other.

## Non-goals

- Changing ADR 010 invariants or adding a ledger type.
- Changing ADR 019 or automatically assigning a PricingProfile.
- Converting, merging, or reassigning Accounts.
- A separate commission wallet or a spendable `total_earned`.
- Cash-out, team-to-team transfer, or spending the settlement Account as the customer's payment method.
- Automatic customer migration between an INDIVIDUAL channel and a TEAM channel.
- Automatic removal, transfer, or repair of an attribution that conflicts with TEAM membership.
- Implementing schema, API, portal, admin, or production changes in this ADR.

## Consequences

A seller can keep one Personal Account for both their own purchases and their individual commission. A team seller keeps a separate Organization Account and the existing membership rules. The portal must ask which channel a multi-context user means, and every request must check that answer again.

The cost is a second owner shape on `PartnerChannel` and a request-scoped context. ADR 023's "exactly one membership channel" rule no longer describes a user who sells both as an individual and through teams.

## Implementation notes

These notes record the local implementation. They do not replace the decisions above. ADR 023 remains the historical TEAM record, including its `partner_channel.team_account_missing` token.

- Portal routes are path-scoped under `/api/v1/partner/channels/{channel_id}/`. `GET /api/v1/partner/contexts/` lists authorized contexts. The path id is identification only. Every request calls `resolve_authorized_partner_context`.
- Settlement is `resolve_partner_settlement_account`. INDIVIDUAL uses the owner's existing Personal Account. TEAM uses the Organization Account. The resolver does not create an Account or change `Account.kind`.
- An INDIVIDUAL grant may spend the full available Personal Account balance. There is no separate commission wallet. `CreditService` remains the only balance mutator.
- Current attribution and TEAM membership activation share one lock order: the User row, then PartnerChannel rows in primary-key order. Either write raises `partner_self_referral` and leaves the other relationship in place. Suspended and revoked membership is not that conflict.
- `PartnerMarginService.accrue` still skips self-referral as `partner_margin.self_referral` without locking Membership. That skip is a defensive backstop for an already-illegal pair. It does not repair the pair.
- A missing or illegal settlement account during accrual logs `reason=partner_margin.settlement_invalid` and `error_type=PartnerSettlementAccountMissing` or `error_type=PartnerChannelOwnershipInvalid`, then rolls the fulfillment transaction back.
- Commission for an INDIVIDUAL channel is credited to the owner's Personal Account. The portal-selected channel does not choose the order-time destination. Attribution does.
- The portal treats more than one context as normal. The last selected channel id is browser state and is reauthorized on each request. `is_active=false` stays visible and stops new accruals.
- INDIVIDUAL channels are admin-provisioned through `create_individual_partner_channel`. There is no self-service activation. After creation, `kind`, `owner_user`, and `organization` stay immutable. `is_active` and `revenue_share_percent` stay editable through the channel-row lock.
- Legacy `/api/v1/orgs/partner/` reads, the legacy invite-link routes, and `POST /api/v1/billing/partner-grants/` stay on their previous semantics until a separate retirement.

## Related

- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — financial owner, ledger, and `CreditService`; not amended here
- [ADR 019](./019-account-pricing-profiles.md) — purchase pricing; not partner commission
- [ADR 020](./020-organization-team-accounts.md) — Organization Account and Membership; still the TEAM owner path
- [ADR 023](./023-partner-channel.md) — historical TEAM channel; parts listed above are amended by this ADR
