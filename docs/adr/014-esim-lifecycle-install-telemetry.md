# ADR 014: eSIM lifecycle and install telemetry

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Amended | **2026-08-14 (reactivate `exhausted`/`expired` on explicit provider `ACTIVE`)** |
| Deciders | Faza 5 Wave 1 design freeze |

## Context

Faza 2 shipped purchase, QR/install payload, on-demand usage, and top-up. Post-purchase UX still dumps users on a raw eSIM detail page. `Esim.status` stays `unused` after provision and is not advanced. There is no install funnel telemetry, no purchase-time activation-policy snapshot, and no sole mutator for product lifecycle state.

Without locked lifecycle rules, status writes will sprawl across views and commands (same failure mode ADR 010 avoided for money via `CreditService`).

## Decision

Adopt a **guided post-purchase setup**, a **guarded `Esim.status` state machine** owned only by `LifecycleService`, an **append-only `EsimLifecycleEvent`** trail for install telemetry, and an **activation-policy snapshot on `Esim`** at fulfillment.

### Product state

Expand `Esim.status` (no parallel `lifecycle_status`). Values:

`purchased`, `installation_started`, `installed`, `activated`, `in_use`, `exhausted`, `expired`, `unknown`.

Migration: existing `unused` → `purchased` via Django `RunPython` only (no ops backfill script).

### State machine

Allowed transitions:

```text
purchased → installation_started → installed → activated → in_use
                                                      ↘ exhausted
                                                      ↘ expired
in_use → exhausted | expired
exhausted → expired
exhausted → activated | in_use   (reactivation; provider ACTIVE only)
expired → activated | in_use     (reactivation; provider ACTIVE only)
purchased → installed   (wizard may skip intermediate telemetry)
```

Forbidden: install-funnel reverse transitions (e.g. `expired → installed`, `expired → purchased`, `in_use → purchased`). Reactivation `exhausted|expired → activated|in_use` is **not** an install-funnel reverse.

**Unknown non-downgrade:** if the provider reports `UNKNOWN`, never set local status to `unknown` when current status is already past `purchased`. Keep current status; optionally append `provider.usage_unknown`. `unknown` is reserved for cold/error bootstrap only. `UNKNOWN` never reactivates.

### Reactivation after top-up (2026-08-14 amend)

A fulfilled top-up (manual or auto) can restore provider data after `exhausted` or `expired`. Sticky terminal `Esim.status` must not hide an active provider plan.

**Sole entry:** only `LifecycleService.apply_provider_usage()` may reactivate, and only when the **current** provider usage payload has `status` equal to `ACTIVE` (trim + case-normalize). Target is `in_use` when that same payload shows consumption, otherwise `activated`.

**Must not** reactivate from:

- cached `Esim.usage_status` / `usage_remaining_mb` / `usage_expired_at` / `usage_synced_at`
- a `Topup` row (`fulfilled` or otherwise)
- the assumption that a purchase or auto-top-up succeeded

Provider `EXPIRED` or `FINISHED` must not reactivate. Repeated identical `ACTIVE` syncs are idempotent (no status change, no extra lifecycle event).

This amend does **not** change [ADR 010](./010-polygon-usdt-prepaid-credits.md). Implementation must not touch billing, ledger, refund, or re-run top-up fulfillment. Reactivation runs on the next normal usage sync (`GET /me/esims/{id}/usage/` or beat). Post-fulfillment usage refresh is **out of scope** for the API PR that implements this amend.

Implementation of this amend is a **separate API PR** after this docs PR merges. Do not implement API, web, or production smoke in this docs PR.

**Sole mutator:** only `LifecycleService` may write `Esim.status`. Architecture test enforces this (same pattern as ADR 010 / `CreditService`).

| Status | Who drives |
|--------|------------|
| `purchased` | `LifecycleService.mark_purchased` on fulfillment |
| `installation_started` / `installed` | Client-attested events → `LifecycleService` |
| `activated` / `in_use` / `exhausted` / `expired` | Provider usage via `LifecycleService.apply_provider_usage` |
| Reactivation `exhausted`/`expired` → `activated`/`in_use` | Same path; **only** explicit provider `ACTIVE` on that payload |
| Provider `UNKNOWN` | Event only; no downgrade; no reactivation |

### Setup session fields on `Esim`

| Field | Purpose |
|-------|---------|
| `setup_version` | Wizard version at last open/complete |
| `setup_resume_step` | 1–4 resume position |
| `setup_completed_at` | Wizard finished |
| `setup_skipped_at` | User dismissed setup |

### Activation policy

- `Package.activation_policy` — synced from Airalo for catalog (`first_usage` \| `installation` \| `unknown`).
- `Esim.activation_policy` — **copied at fulfillment** from the package. Wizard/support always read the eSIM snapshot, never the live catalog row.

### `EsimLifecycleEvent` (append-only)

| Field | Notes |
|-------|--------|
| `id` | UUID PK |
| `esim`, `user` | FKs |
| `event_type` | Namespaced string |
| `source` | `client` \| `system` \| `provider` |
| `schema_version` | Payload contract version (start at `1`) |
| `idempotency_key` | Unique per eSIM; retries return existing row |
| `setup_session_id` | Correlates one onboarding run |
| `payload` | JSON |
| `user_agent` | Optional |
| `created_at` | Immutable |

Never update or delete event rows.

### Event taxonomy

Wave 1 implements `install.*` plus minimal `provider.*` / `system.*`. Reserved for later: `usage.*`, `topup.*`, `support.*`, `notification.*`.

Client allowlist (Wave 1): `install.opened`, `install.qr_rendered`, `install.qr_zoomed`, `install.apple_install_clicked`, `install.manual_install_clicked`, `install.completed`, `install.roaming_checklist_viewed`, `install.setup_confirmed`, `install.setup_skipped`.

### API

- `POST /api/v1/me/esims/{id}/events/` — JWT + ownership; idempotent via `idempotency_key`.
- `GET /api/v1/me/esims/{id}/events/` — chronological trail for owned eSIM.
- Cross-user access matches list/detail (`404`). Never accept body `user_id` / foreign `esim_id` to override path ownership.

Additive fields and new endpoints do not bump `/api/v1`. Renaming status `unused` → `purchased` is a **coordinated v1 sole-consumer exception** (OpenAPI changelog + `roamkit-web` same wave). Do not open `/api/v2` for this rename.

### Domain events (ADR 005)

Publish past-tense facts for server transitions (`EsimInstalled`, `EsimActivated`, `EsimFirstUsageDetected`, …). No real transactional email in Wave 1.

### Observability

Countable signals from day one (structured logs and/or counters): `setup_started`, `setup_completed`, `setup_skipped`, `activation_detected`, `purchase_to_activation_seconds`.

## Sequence diagrams

### Install flow

```mermaid
sequenceDiagram
  participant U as User
  participant W as roamkit-web
  participant A as roamkit-api

  U->>W: Purchase succeeds
  W->>W: Redirect to /me/esims/id/setup
  W->>A: POST install.opened
  A->>A: LifecycleService installation_started
  U->>W: Scan QR or Apple install
  W->>A: POST install.completed
  A->>A: LifecycleService installed
  U->>W: Confirm setup
  W->>A: POST install.setup_confirmed
```

### Activation flow

```mermaid
sequenceDiagram
  participant U as User
  participant Device as Phone
  participant P as Provider
  participant A as roamkit-api

  U->>Device: Install and enable eSIM
  Device->>P: Profile download / first attach
  U->>A: GET usage or later poll
  A->>P: get_usage
  P-->>A: status ACTIVE
  A->>A: LifecycleService activated
```

### Usage sync flow

```mermaid
sequenceDiagram
  participant W as roamkit-web
  participant A as roamkit-api
  participant P as Provider

  W->>A: GET /me/esims/id/usage/
  A->>P: get_usage
  P-->>A: remaining total status expired_at
  A->>A: UsageService updates usage cache
  A->>A: LifecycleService.apply_provider_usage
  Note over A: UNKNOWN never downgrades local status
  Note over A: ACTIVE may reactivate exhausted or expired
```

## Consequences

### Positive

- Product lifecycle has one mutator and testable guards.
- Install funnel is measurable without a notification engine.
- Historical purchases keep activation rules after catalog changes.

### Negative

- Client-attested `installed` can be wrong; support relies on event trail + provider usage.
- Sole-consumer enum rename still requires coordinated web deploy.
- Until the API PR ships, `Esim.status` can stay `expired`/`exhausted` while provider usage is already `ACTIVE` (cached remaining data and package expiry may look live).

## Stop rule

Now that this ADR is **Accepted** (as amended — reactivate on explicit provider `ACTIVE`):

- Do **not** start the API implementation PR until this docs amend is merged.
- Do **not** reactivate except via `LifecycleService.apply_provider_usage` on an explicit current-payload `ACTIVE`.
- Do **not** treat cached usage fields, a fulfilled `Topup`, or a successful purchase as sufficient to change `Esim.status`.
- Do **not** allow `expired → installed` or `expired → purchased`.
- Do **not** amend or bypass [ADR 010](./010-polygon-usdt-prepaid-credits.md): no billing, ledger, refund, or fulfillment side effects in the lifecycle PR.
- Do **not** couple this amend to a post-fulfillment usage refresh.

Changing this Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.

## Related

- [ADR 005](./005-domain-events.md)
- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — sole-mutator pattern; **unchanged** by this amend
- [RFC 002](../rfcs/002-post-purchase-onboarding.md)
- [API versioning](../../standards/api-versioning.md)
