# Partner channel rollout (ADR 024)

Operational sequence for shipping individual and team partner channels. This does not deploy anything.

ADR 024 is the current decision. ADR 023 remains the historical team-channel record.

API and web are separate repositories. Staging deploys from `develop`. Production deploys from `main`. Docs do not have to ship in the same step as either app.

## Order

1. Deploy the API that contains migration `0012_partner_channel_kind` and the path-scoped `/api/v1/partner/` routes.
2. Confirm the migration committed and `partner_integrity_check` is clean on that database.
3. Smoke the new routes against the old web still calling the legacy `/api/v1/orgs/partner/` and `/api/v1/billing/partner-grants/` URLs. Those routes stay compatible.
4. Deploy the web build that calls only the path-scoped partner routes.
5. Docs can ship before or after the apps. They do not change runtime behavior.

Do not deploy the new web before the API that serves `/api/v1/partner/contexts/` and `/api/v1/partner/channels/{channel_id}/`.

## Migration preflight

`0012_partner_channel_kind` is read-only in its preflight. It refuses when a current customer of a team channel is also an active member of that organization. It does not delete the attribution, change the membership, or move money.

If migrate refuses:

- read the error for the listed rows
- decide the pair outside the migration
- do not expect the migration to repair it
- run migrate again only after the illegal pair is gone

## Operator checklist

Do not run this against production from a development checkout.

### Before deploy

- [ ] Take the database backup the current production checklist already requires before a schema migration.
- [ ] Record `django_migrations` for `billing` and confirm the database is at `0011_partner_invite_bonus`.
- [ ] On a copy of production data, look for a current customer of a team channel who is also an active member of that organization. `0012` will refuse that pair and will not repair it.
- [ ] Know `PARTNER_CHANNEL_ENABLED` for the target environment.
- [ ] API image contains migration `0012_partner_channel_kind` and `/api/v1/partner/`.
- [ ] Web image calls only `/api/v1/partner/contexts/` and `/api/v1/partner/channels/{channel_id}/`.

### Deploy the API first

- [ ] Deploy the API. Do not deploy the new web yet.
- [ ] Run migrations. `0012` completes, or it aborts with the database still at `0011` and no repaired rows.
- [ ] Run `python manage.py partner_integrity_check`. Expect `partner_integrity_check=clean` and exit 0.
- [ ] API health is green.
- [ ] `GET /api/v1/partner/contexts/` returns 200 for an authenticated user.
- [ ] One existing team partner can read summary, customers, grant history, and the invite link on the new path.
- [ ] A user with more than one context receives every context. The new routes do not return `partner_context_ambiguous`.
- [ ] A viewer receives 403 `partner_grant_forbidden` and `partner_invite_forbidden` on writes.
- [ ] The previous web build still works against the legacy partner URLs.

### Deploy the web

- [ ] Deploy the web only after the checks above.
- [ ] The portal loads contexts.
- [ ] One context opens without a picker. Several contexts restore the last browser choice or ask for one.
- [ ] Switching context replaces summary, customers, grants, and the invite link.
- [ ] The browser network log for the portal shows no `/api/v1/orgs/partner/` or `/api/v1/billing/partner-grants/` calls.

### After both deploys

- [ ] A user with no partner channel can still sign in, see a personal balance, and buy or top up an eSIM.
- [ ] A team purchase credits the organization account. An individual purchase of an attributed customer credits that owner's personal account.
- [ ] A team grant and an individual grant move money once and replay the same idempotency key without a second movement.
- [ ] An owner can regenerate the invite link. A viewer cannot.
- [ ] An inactive channel remains visible and does not accrue a new margin.
- [ ] `partner_integrity_check` is clean.
- [ ] Logs show `partner.grant.succeeded` or `partner_margin.accrued` for the smoke actions, and `partner.legacy_endpoint.used` only if something still calls a legacy URL.

## Before deploy

- Take the database backup the current production checklist already requires before a schema migration.
- Run the API test suite on the release commit.
- After migrate, run `python manage.py partner_integrity_check`. A clean database prints `partner_integrity_check=clean` and exits 0. Any violation exits 1 and changes nothing.

## After deploy

- A user with no partner channel can still buy and manage personal eSIMs.
- An existing team owner still sees that team's summary, customers, grants, and invite link.
- An individual partner sees one individual context and the same pages for that channel.
- A user with an individual channel and a team channel, or two teams, must choose. The portal does not treat that as an error.
- A viewer can read and cannot grant or change the invite link.
- A team admin can grant and cannot change the invite link.
- An owner can grant and manage the invite link.
- An inactive channel stays readable and does not accrue new margin.
- A team member, and a suspended or revoked membership, has no portal context.

## Pending attribution during activation

Email activation applies a pending partner attribution inside the same transaction as setting the password. If that pending row would make the person a customer of a team they already belong to, or of an individual channel they own, activation still completes. The pending row stays until its existing expiry and is not turned into a current attribution. No registration bonus is credited. After expiry, the normal pending cleanup removes it. The person can buy and manage personal eSIMs without a partner attribution. Nothing retries that pending row after the account is active.

## Rollback

| State | API rollback | Database | Web rollback |
| --- | --- | --- | --- |
| `0012` not applied | Normal, back to the previous API image | Not involved | Normal. The old web matches the old API |
| `0012` applied, every channel still a legacy team row | Possible. Old code still finds an organization on every channel | Reverse is allowed only if the migration guard still sees legacy team rows | Possible. Legacy partner URLs remain |
| `0012` applied and an individual channel exists | Not safe. Old code does not understand `kind=individual` or a null organization | Do not reverse. Reverse refuses and does not delete the channel | Possible. Legacy URLs remain, but they do not serve the individual channel |

New API with the old web stays compatible. Old API with the new web does not: the new web calls `/api/v1/partner/`, which the old API does not serve. Deploy the API, verify it, then deploy the web.

### Web

The previous web build calls the legacy partner URLs. Those URLs remain on the new API. Rolling the web back is safe while the new API is still serving them.

### API application

Old API code does not understand `kind=individual` or a null organization. It is safe to roll the API process back only while every `PartnerChannel` row is still a legacy team row (`kind=team`, `owner_user` null, organization set).

Once any individual channel exists, do not roll the API back to a build from before ADR 024. Those rows would be read as team channels with no organization.

### Database

`0012` reverses only when every channel is still a legacy team row. Reverse refuses if an individual channel exists, and it does not delete channels to make the downgrade succeed.

After the first individual channel is created, schema rollback is not available. Application rollback of the API is not available either. Web rollback to the legacy client remains available because the legacy routes stay.
