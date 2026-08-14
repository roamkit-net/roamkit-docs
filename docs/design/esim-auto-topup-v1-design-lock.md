# eSIM Auto Top-up v1 — Design Lock

| Field | Value |
|-------|-------|
| Status | **Accepted** (architecture freeze for implementation) |
| Date | 2026-08 |
| Scope | Product auto top-up for owned eSIMs (v1) |
| Related | [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md), [ADR 012](../adr/012-billing-extensibility-rules.md), [ADR 004](../adr/004-provider-interfaces.md), [ADR 005](../adr/005-domain-events.md), [ADR 014](../adr/014-esim-lifecycle-install-telemetry.md) |
| Pattern | **Design Lock → PR2–PR5 implementation** (no redesign mid-flight) |

This document locks Auto Top-up v1 so PR2–PR5 implement an accepted design.

**Shipped.** Multi-condition OR triggers are a **separate epic** — see [eSIM Auto Top-up v2 — Design Lock](./esim-auto-topup-v2-design-lock.md). Do not reopen this v1 lock to retrofit OR semantics.

**Do not reopen locked decisions** unless: (a) bug fix, (b) docs inconsistency, or (c) a real architectural blocker. Everything else goes to a **separate backlog** item — not into the active implementation PR.

This design **does not** amend ADR 010 financial invariants (`balance >= 0`, `CreditService` sole mutator, append-only ledger).

### Locked v1 checklist (implementers)

Everything below is **in scope for PR2–PR5**. Do not drop items; do not add product features outside this list.

| Item | Locked detail |
|------|----------------|
| `cooldown_until` | Set on successful auto top-up; default 15 min (`AUTO_TOPUP_COOLDOWN_SECONDS=900`); beat skips trigger while active |
| Usage freshness | Never buy on stale cache; max age 10 min; refresh first; skip if refresh fails |
| `status` + `reason` | Coarse status (`active` / `paused` / `blocked` / `disabled`) + separate reason codes |
| Rollout | `AUTO_TOPUP_ENABLED` + mode `off` → `staff` → `allowlist` → `percent` → `all` |
| `version` / `If-Match` | Optimistic concurrency on policy mutate; stale write → 409 |
| Domain events | `AutoTopupSucceeded`, `AutoTopupPolicyCreated`, `AutoTopupPolicyUpdated` (+ funds-pause CTA path) |
| Admin support fields | Read-only: last trigger, status, reason, last Topup, last idempotency key, `cooldown_until` |
| Also required | `minimum_age`; provider timeout ≠ pause; idempotency layered with cooldown/freshness; Prometheus metrics; DoD test matrix |

---

## One-sentence product goal

> When an eSIM’s usage or expiry hits a user-chosen rule, RoamKit buys **one package from that eSIM’s Available top-ups** using prepaid credits — safely, once per trigger epoch, behind a rollout flag.

---

## Locked separation: Auto Top-up ≠ Subscription

| Concern | Owner | Mechanism |
|---------|-------|-----------|
| Calendar prepaid renew | `billing.Subscription` + `SubscriptionService` | `next_billing_date` (ADR 010) |
| Usage / expiry reaction | **eSIM domain** `EsimAutoTopupPolicy` + `AutoTopupService` | Usage cache + beat |

Auto top-up **must not** extend or reuse `Subscription` / `SubscriptionService` for triggers.

Spend still goes through existing `TopupService.purchase` → `CreditService.debit` → provider `submit_topup`. Providers keep **zero** billing imports (ADR 004 / ADR 010).

---

## Constraint: Available top-ups only

Packages come **only** from `TopupService.list_topups(esim)` for that ICCID (the same list as `/me/esims/{id}` “Available top-ups”).

| Layer | Rule |
|-------|------|
| Save policy | `package_id` must appear in current `list_topups(esim)`; else 400 |
| UI | Enable auto top-up only for a row from that list — never invent packages |
| Beat execute | Call `TopupService.purchase`; if package gone → `status=blocked`, `reason=package_unavailable` |
| Unlimited | Allowed for `expiry` only; reject `usage_zero` / `usage_threshold` when package or live usage is unlimited |

---

## v1 in scope

- Model `EsimAutoTopupPolicy` (account + esim owned like `Topup`)
- Celery beat / worker evaluation
- Triggers: `usage_zero` \| `usage_threshold` (configurable MB, default 500) \| `expiry`
- Modes: `until_funds` \| `fixed_count`
- UI on Available top-ups (no separate page)
- Idempotent purchase via existing `TopupService` + `CreditService`
- Reliability guardrails (below)
- Policy `version` / `If-Match` concurrency
- Domain events for success + policy create/update
- Prometheus metrics
- Support-oriented Django admin (read-only ops fields)
- Feature-flag rollout

### Explicitly out of v1 (backlog)

- Always ask / notify-before-buy
- Auto Switch package ladder
- Monthly budget / max USDT per month
- Credit limit / overdraft (requires future ADR amending ADR 010 `balance >= 0`)

**Credit limit sequencing:** ship auto top-up → stabilize in production → only then ADR for overdraft. Not parallel with v1.

---

## Domain model

`EsimAutoTopupPolicy` (`apps.esims`):

| Field | Notes |
|-------|-------|
| `account` | FK → `billing.Account` |
| `esim` | FK → `esims.Esim` |
| `package_id` | Provider external id from Available top-ups |
| `enabled` | User intent |
| `status` | `active` \| `paused` \| `blocked` \| `disabled` |
| `reason` | `insufficient_funds` \| `package_unavailable` \| `usage_unknown` \| `provider_error` \| `manual_pause` \| `count_exhausted` \| null when healthy |
| `trigger_mode` | `usage_zero` \| `usage_threshold` \| `expiry` |
| `threshold_mb` | Required when `usage_threshold` |
| `renew_mode` | `until_funds` \| `fixed_count` |
| `remaining_count` | Required when `fixed_count`; decrement after successful purchase |
| `cooldown_until` | Set on success (default now + 15 min) |
| `last_triggered_at` | |
| `last_topup_id` | Nullable FK → `Topup` |
| `last_idempotency_key` | Support |
| `version` | Integer; bump on user/admin mutate; optimistic concurrency |

Constraints: one enabled policy per `esim` in v1; CHECKs on threshold/count as appropriate.

**Status vs reason:** keep status coarse; add reasons without status-value migrations.

---

## Settings (defaults locked)

| Setting | Default | Role |
|---------|---------|------|
| `AUTO_TOPUP_ENABLED` | `false` | Master gate |
| `AUTO_TOPUP_COOLDOWN_SECONDS` | `900` | Post-success cooldown |
| `AUTO_TOPUP_USAGE_MAX_AGE_SECONDS` | `600` | Max usage cache age before buy |
| `AUTO_TOPUP_MINIMUM_AGE_SECONDS` | `600` | Skip until eSIM activation/install age ≥ this |
| `AUTO_TOPUP_ROLLOUT_MODE` | `off` | `off` \| `staff` \| `allowlist` \| `percent` \| `all` |
| `AUTO_TOPUP_ALLOWLIST_ACCOUNT_IDS` | `[]` | When mode=`allowlist` |
| `AUTO_TOPUP_ROLLOUT_PERCENT` | `0` | 0–100 when mode=`percent` |

---

## Beat evaluation flow

```text
Beat tick
  → Rollout gate (master + mode)
  → Refresh usage if stale / missing
  → Fresh enough? else skip (never buy on stale/failed refresh)
  → In cooldown? else skip trigger (usage refresh only)
  → Past minimum_age? else skip
  → status == active?
  → Trigger met?
  → TopupService.purchase (idempotent key)
       → success: cooldown + events + metrics + maybe decrement count
       → InsufficientFunds → status=paused, reason=insufficient_funds
       → package missing → status=blocked, reason=package_unavailable
       → provider timeout / transient → no status change; retry next beat
```

### Must-have guardrails

1. **Cooldown** after success — provider usage lag must not cause double buy
2. **Usage freshness** — never purchase on stale cache; refresh first; skip if refresh fails
3. **`status` + `reason`** split
4. **Rollout** — off → staff → allowlist → percent → all

Also required: **minimum_age**; provider timeout ≠ pause; layered **idempotency** (does not replace cooldown/freshness).

### Idempotency keys (v1)

- Usage triggers: `auto-topup:{policy_id}:{trigger_mode}:{usage_synced_at_iso}`
- Expiry: `auto-topup:{policy_id}:expiry:{usage_expired_at_iso}`

Passed as `Topup.idempotency_key`. Ledger debit keys remain `topup-debit:{topup.pk}` (unchanged money path). Concurrent beats: `select_for_update` on policy + unique idempotency → one top-up.

---

## Domain events (snapshots)

| Event | Purpose |
|-------|---------|
| `AutoTopupSucceeded` | `policy_id`, `topup_id`, `package_id`, `amount`, `remaining_count`, account/esim ids, ledger refs, `event_version` |
| `AutoTopupPolicyCreated` | Who/what enabled policy (support / analytics) |
| `AutoTopupPolicyUpdated` | Policy mutations with `version`, `actor` when available |
| Funds-pause style event | Deposit CTA when `insufficient_funds` (mirror existing billing pause patterns) |

Happy-path handlers must not re-fetch core snapshot fields (ADR 005).

---

## Metrics (Prometheus)

- `auto_topup_attempts_total`
- `auto_topup_success_total`
- `auto_topup_failed_total` (labels: reason)
- `auto_topup_paused_total` (labels: reason)
- `auto_topup_duration_seconds`

---

## API

- `GET/PUT/DELETE /api/v1/me/esims/{id}/auto-topup/`
- PUT: package, trigger, threshold?, renew mode, count?; concurrency via `version` and/or `If-Match` → **409** on stale write
- Validate `package_id` against `list_topups` on save
- Response includes `status`, `reason`, `cooldown_until`, `version`
- Manual `POST …/topups/` unchanged

---

## Web UI

On **Available top-ups** (`/me/esims/{id}`):

- Enable auto top-up for a selected package row
- Trigger: expired / remaining &lt; N MB (default 500) / remaining = 0
- Mode: until funds / renew N times
- Banner from `status` + `reason`
- Existing Buy / Add USDT shortfall UX unchanged

---

## Admin (support)

Read-only emphasis: last trigger, status, reason, last Topup, last idempotency key, `cooldown_until`. No direct balance edits (still `CreditService` only).

---

## DoD test matrix (implementation)

| Scenario | Expected |
|----------|----------|
| usage=0, fresh | one top-up |
| usage=0 + stale cache, refresh fails | no top-up |
| provider timeout | retry next beat; policy stays active |
| package removed | blocked + `package_unavailable` |
| insufficient funds | paused + `insufficient_funds` |
| fixed_count=1 | buy once then pause/disable + `count_exhausted` |
| threshold=500 | fires once per trigger epoch; cooldown blocks immediate rebuy |
| cooldown active | no second top-up |
| two beats in parallel | one top-up |
| eSIM younger than minimum_age | skip |
| rollout off / not allowlisted | skip |

---

## Feature-flag rollout (ops)

```text
AUTO_TOPUP_ENABLED=false (default)
  → staff only
  → allowlist accounts
  → percent (e.g. 10%)
  → all (100%)
```

Validate in production without broad exposure before `all`.

---

## Implementation PR slice

| PR | Repo | Scope |
|----|------|--------|
| **PR1** | `roamkit-docs` | This design lock (docs only) |
| PR2 | `roamkit-api` | Schema + model + migration + support admin |
| PR3 | `roamkit-api` | `AutoTopupService` + beat + guardrails + events + metrics + DoD tests |
| PR4 | `roamkit-api` | Me API + version/`If-Match` + OpenAPI + rollout enforcement |
| PR5 | `roamkit-web` | Available top-ups UI + banner + e2e |

Never mix schema + UI in one PR. After this lock is merged: implement; do not redesign inside PR2–PR5.

---

## Architecture guardrails (non-negotiable)

1. Providers never touch money.
2. Only `CreditService` mutates balances/ledger (via existing top-up purchase).
3. Do not reuse `SubscriptionService` for usage triggers.
4. Idempotency + cooldown + freshness are layered; none replaces the others.
5. Never purchase on stale or failed usage refresh.
6. Provider timeout ≠ pause.
7. ADR 010 `balance >= 0` unchanged in v1.
