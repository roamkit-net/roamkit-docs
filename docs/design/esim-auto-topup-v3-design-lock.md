# eSIM Auto Top-up v3 — Design Lock

| Field | Value |
|-------|-------|
| Status | **Accepted** (architecture freeze for implementation) |
| Date | 2026-08 |
| Scope | Optional policy lifetime (`active_until`) for owned eSIM auto top-up (v3) |
| Related | [v1 design lock](./esim-auto-topup-v1-design-lock.md), [v2 design lock](./esim-auto-topup-v2-design-lock.md), [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md), [ADR 012](../adr/012-billing-extensibility-rules.md), [ADR 005](../adr/005-domain-events.md) |
| Pattern | **Design Lock → PR2–PR5 implementation** (no redesign mid-flight) |
| Supersedes | Nothing in v1/v2. **Additive** lifetime guard only |

This document locks Auto Top-up **v3** so PR2–PR5 implement an accepted design.

**Do not reopen locked decisions** unless: (a) bug fix, (b) docs inconsistency, or (c) a real architectural blocker. Everything else goes to a **separate backlog** item — not into the active implementation PR.

This design **does not** amend ADR 010 financial invariants (`balance >= 0`, `CreditService` sole mutator, append-only ledger).

v1 and v2 remain shipped capabilities ([capability-status](../ops/capability-status.md)). v3 is a **new epic**, not a hotfix of Accepted v1/v2 locks. Inherited unless this document says otherwise: Available-topups-only, spend via `TopupService` → `CreditService`, v2 trigger shape (`expiry_enabled` + `usage_mode`), renew modes (`until_funds` \| `fixed_count`), cooldown, freshness, minimum age, rollout, admin read-only support fields, one policy per eSIM.

### Locked v3 checklist (implementers)

Everything below is **in scope for PR2–PR5**. Do not drop items; do not add product features outside this list.

| Item | Locked detail |
|------|----------------|
| Lifetime field | Optional `active_until` (`timestamptz` UTC, nullable) |
| Null semantics | `null` = no date limit (current behaviour) |
| Guard | Beat: if `active_until` set and `now >= active_until` → pause + `schedule_ended`; no purchase |
| Not a renew mode | Does **not** replace or merge with `until_funds` / `fixed_count` |
| Status / reason | Reuse `status=paused` + new reason `schedule_ended` (no new status enum) |
| Cooldown | Changing `active_until` alone does **not** reset `cooldown_until` |
| API | Additive `active_until` on GET/PUT (nullable ISO-8601); not breaking |
| UI | Date control **outside** Renew mode section; empty = no limit |
| UTC mapping | UI calendar date **D** → store exclusive bound start of **D+1** 00:00 UTC |
| Resume | PUT with future `active_until` (or null) while `reason=schedule_ended` → `status=active`, clear reason (if otherwise valid) |

---

## One-sentence product goal

> Let users keep existing renew modes and triggers, and optionally stop auto top-up after a chosen calendar day — without replacing “until funds” or “fixed count”.

---

## Why (three intents)

| Intent | Mechanism |
|--------|-----------|
| Do not run out of data while travelling | Triggers + `until_funds` |
| Do not spend more than N purchases | `fixed_count` + `remaining_count` |
| Do not keep auto top-up after returning home | **`active_until` (this epic)** |

`fixed_count` solves only spend-count. `active_until` solves only schedule end. They are **orthogonal**.

**Do not** replace `fixed_count` with a date. That would lose parent/enterprise purchase caps and test rollouts.

---

## Relationship to v1 / v2

| Topic | Rule |
|-------|------|
| v1 / v2 design locks | Remain Accepted; do not reopen to fold lifetime into renew mode |
| Trigger shape | Unchanged (v2) |
| Renew mode | Unchanged (`until_funds` \| `fixed_count`) |
| Money / Subscription | Unchanged — still ≠ `billing.Subscription`; still ADR 010 |
| Me API | **Additive** field under same path `/api/v1/me/esims/{id}/auto-topup/` |

---

## Locked lifetime model

```text
active_until: timestamptz | null   # UTC; null = no schedule limit
```

### Evaluation conjunction (locked)

A successful auto top-up requires **all** of:

```text
policy status == active
AND enabled
AND (active_until is null OR now < active_until)
AND renew_mode permits (until_funds | remaining_count > 0)
AND trigger met (v2 OR semantics)
AND inherited guards (rollout, freshness, cooldown, minimum_age, …)
```

### Must-lock semantics

#### 1. Early schedule guard (before purchase)

On beat evaluation, after load/`select_for_update` and before trigger fire / `TopupService.purchase`:

```text
if policy.active_until is not null and now >= policy.active_until:
    status = paused
    reason = schedule_ended
    return  # no purchase
```

Mirror `count_exhausted`: coarse status stays **`paused`**; reason carries the why. **Do not** add a new `Status` value.

#### 2. UTC date mapping (UI → API)

User-facing control is a **calendar date** (no time-of-day picker in v3).

| UI value | Persisted `active_until` |
|----------|--------------------------|
| empty | `null` |
| date **D** | `D + 1 day` at `00:00:00.000 UTC` (exclusive end) |

So “Policy active until 20 Aug 2026” remains eligible through **end of 20 Aug UTC**, and ends at the first instant of **21 Aug UTC**.

Beat / API always compare with timezone-aware UTC `now`.

**Out of v3:** per-user local timezone calendars (backlog). UTC is locked.

#### 3. Orthogonal to renew mode

| Config | Meaning |
|--------|---------|
| `until_funds` + `active_until=null` | Keep going while credits allow |
| `until_funds` + `active_until` set | Keep going while credits allow **and** before schedule end |
| `fixed_count` + `active_until=null` | At most N purchases |
| `fixed_count` + `active_until` set | At most N purchases **and** before schedule end |

Whichever limit hits first wins (`count_exhausted` vs `schedule_ended`).

#### 4. Cooldown / trigger-config

Changing `active_until` alone does **not** clear `cooldown_until`.

Cooldown reset remains the v2 rule: only when `(expiry_enabled, usage_mode, threshold_mb)` changes.

#### 5. PUT validation

| Case | Behaviour |
|------|-----------|
| `enabled=true` and `active_until` in the past (`now >= active_until`) | **400** |
| `enabled=false` and past `active_until` | Allowed (stored; no beat fire while disabled) |
| `active_until=null` | Allowed |
| Resume: was `paused` + `schedule_ended`, PUT sets null or future `active_until` | Set `status=active`, clear `reason` (subject to other validity checks) |

Inherited: PUT during in-flight auto purchase → **409**.

#### 6. UI placement

Renew mode stays:

```text
Renew mode
○ Until credits run out
○ Fixed number of renewals
   [Remaining renewals]
```

Lifetime is a **separate** control (not inside renew radios):

```text
Policy active until
[ 20 Aug 2026 ]
(leave empty = no date limit)
```

Banner when `reason=schedule_ended`: copy equivalent to “Paused — policy end date reached.”

---

## Explicitly out of v3 (separate epics)

- Replacing `fixed_count` / `until_funds` with a date-only renew mode
- User-local timezone for the calendar date
- Time-of-day picker / “ends at 15:00”
- Monthly budget / max USDT per month
- Always ask / notify-before-buy
- Auto Switch package ladder
- Multiple policies per eSIM
- Credit limit / overdraft (still requires future ADR amending ADR 010)
- New status enum values

---

## Domain model changes (relative to v2)

Keep `EsimAutoTopupPolicy`. Add:

| Field | Notes |
|-------|-------|
| `active_until` | `DateTimeField(null=True, blank=True)`; UTC exclusive bound |

Unchanged: triggers, renew mode, count, cooldown, status enum, support fields, `version`.

Reason enum: add `schedule_ended`.

Constraints: still **one policy per `esim`**.

---

## Beat evaluation (v3 addition)

```text
Beat tick
  → Rollout / freshness / cooldown / minimum_age / status==active (as v1/v2)
  → Schedule guard:
       if active_until and now >= active_until
         → pause + reason=schedule_ended; stop
  → Select fire reason (v2)
  → TopupService.purchase …
```

---

## Domain events

Inherited v2 events.

On schedule end (beat pause path): prefer existing funds-pause style / policy-updated emission pattern used for `count_exhausted` / `insufficient_funds` — include `reason=schedule_ended` in snapshot when an event is published. No new event type required for v3 DoD.

Optional (not a PR1 merge blocker): include `active_until` on `AutoTopupPolicyCreated` / `AutoTopupPolicyUpdated` snapshots.

---

## Metrics (Prometheus)

Inherited counters, plus pause label:

- `auto_topup_paused_total{reason=schedule_ended}` when the schedule guard pauses a policy

No new success reason labels (purchase reasons stay v2 trigger reasons).

---

## API (additive)

Path unchanged: `GET/PUT/DELETE /api/v1/me/esims/{id}/auto-topup/`

| Direction | Body / response |
|-----------|-----------------|
| Write / read | Add optional `active_until` (`string` date-time ISO-8601 UTC, or `null`) |
| Unchanged | v2 triggers, package, enabled, renew mode, count, status, reason, cooldown, version, If-Match → **409** |

OpenAPI updated in PR4. Not a breaking change for clients that ignore unknown fields; web must send/read the field.

GET may return exclusive-bound datetime; web may display calendar date **D** as `active_until_date = (active_until as UTC date) - 1 day` when mapping back for the date input (document in PR5).

---

## Web UI

On Available top-ups (`/me/esims/{id}`), in addition to v2 controls:

```text
Policy active until
[ date ]
Leave empty for no end date.
```

- Separate from Renew mode
- Empty clears `active_until`
- Client maps date ↔ exclusive UTC bound per this lock
- Show schedule-ended banner from `status` + `reason`

---

## Migration (PR2)

- Add nullable `active_until` column (default `NULL` for all existing rows)
- Add `schedule_ended` to reason choices
- No data backfill required
- Reversible: drop column / remove choice in reverse migration when safe

---

## DoD test matrix (v3 additions)

| Scenario | Expected |
|----------|----------|
| `active_until=null`, trigger met | purchase (unchanged) |
| `now < active_until`, trigger met | purchase |
| `now >= active_until`, was active | no purchase; `paused` + `schedule_ended`; metric paused |
| next beat after schedule end | stay paused; no purchase |
| `fixed_count` hits 0 before `active_until` | `count_exhausted` (not schedule) |
| `active_until` hits before count exhausts | `schedule_ended` |
| PUT past `active_until` while enabling | 400 |
| PUT future/null while `schedule_ended` | resume `active` |
| PUT only `active_until` while in cooldown | cooldown unchanged |
| OpenAPI / GET | `active_until` present; v2 fields intact |

Plus inherited v1/v2 DoD.

---

## Implementation PR slice

| PR | Repo | Scope |
|----|------|--------|
| **PR1** | `roamkit-docs` | This design lock (docs only) |
| PR2 | `roamkit-api` | Schema + migration + admin field + reason choice |
| PR3 | `roamkit-api` | Service schedule guard + pause path + metrics + DoD tests |
| PR4 | `roamkit-api` | Me API + OpenAPI + validation/resume + contract tests |
| PR5 | `roamkit-web` | Date control outside Renew + banner + e2e |

Never mix schema + UI in one PR. After this lock is merged: implement; do not redesign inside PR2–PR5.

---

## Architecture guardrails (non-negotiable)

1. Providers never touch money.
2. Only `CreditService` mutates balances/ledger (via existing top-up purchase).
3. Do not reuse `SubscriptionService` for usage triggers.
4. Idempotency + cooldown + freshness + schedule are layered; none replaces the others.
5. Never purchase on stale or failed usage refresh.
6. Provider timeout ≠ pause.
7. ADR 010 `balance >= 0` unchanged.
8. Do **not** replace `fixed_count` with `active_until`.
9. No new status enum — `paused` + `schedule_ended` only.
10. UTC exclusive bound only in v3 (no local-TZ calendars).
