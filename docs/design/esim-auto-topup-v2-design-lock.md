# eSIM Auto Top-up v2 — Design Lock

| Field | Value |
|-------|-------|
| Status | **Accepted** (architecture freeze for implementation) |
| Date | 2026-08 |
| Scope | Multi-condition triggers for owned eSIM auto top-up (v2) |
| Related | [v1 design lock](./esim-auto-topup-v1-design-lock.md), [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md), [ADR 012](../adr/012-billing-extensibility-rules.md), [ADR 004](../adr/004-provider-interfaces.md), [ADR 005](../adr/005-domain-events.md) |
| Pattern | **Design Lock → PR2–PR5 implementation** (no redesign mid-flight) |
| Supersedes | v1 **trigger shape only** (`trigger_mode` → `expiry_enabled` + `usage_mode`) |

This document locks Auto Top-up **v2** so PR2–PR5 implement an accepted design.

**Do not reopen locked decisions** unless: (a) bug fix, (b) docs inconsistency, or (c) a real architectural blocker. Everything else goes to a **separate backlog** item — not into the active implementation PR.

This design **does not** amend ADR 010 financial invariants (`balance >= 0`, `CreditService` sole mutator, append-only ledger).

v1 remains a shipped capability ([capability-status](../ops/capability-status.md)). v2 is a **new epic**, not a hotfix of the Accepted v1 lock. Inherited from v1 unless this document says otherwise: Available-topups-only, spend via `TopupService` → `CreditService`, status/reason codes, renew modes, rollout flags, freshness, minimum age, admin read-only support fields, one policy per eSIM.

### Locked v2 checklist (implementers)

Everything below is **in scope for PR2–PR5**. Do not drop items; do not add product features outside this list.

| Item | Locked detail |
|------|----------------|
| Trigger model | `expiry_enabled` + `usage_mode` (`disabled` \| `threshold` \| `zero`) + `threshold_mb`; **OR** semantics |
| `usage_mode=zero` ≠ `threshold=0` | Distinct reason / metric label / idempotency reason; do not merge code paths |
| Reason precedence | Same beat: expiry wins; **no pending usage queue** after expiry fire |
| Cooldown reset | Changing `(expiry_enabled, usage_mode, threshold_mb)` clears `cooldown_until` |
| Metric | `auto_topup_trigger_reason_total{reason=…}` on success |
| UI OR helper | When both conditions selected: “any selected condition is met” |
| Breaking me API | Remove `trigger_mode`; OpenAPI + contract tests prove absence |
| Migration | 100% forward map from v1 `trigger_mode`; documented rollback |
| Concurrent PUT | If auto-topup purchase in flight for eSIM → PUT **409** (retry later) |
| Inherited v1 | Cooldown duration, freshness, status/reason, rollout, `version`/`If-Match`, spend path, admin fields |

---

## One-sentence product goal

> When an eSIM’s **expiry and/or usage** hits user-chosen conditions (any selected), RoamKit buys **one package from that eSIM’s Available top-ups** using prepaid credits — safely, once per reason epoch, behind a rollout flag.

---

## Relationship to v1

| Topic | Rule |
|-------|------|
| v1 design lock | Remains Accepted historical freeze for shipped v1; do not reopen to “patch in” OR triggers |
| Trigger shape | **Superseded** by this document |
| Money / Subscription separation | Unchanged — still ≠ `billing.Subscription`; still ADR 010 |
| Me API | Breaking replace of fields under same path `/api/v1/me/esims/{id}/auto-topup/` (no URL version bump) |

---

## Locked trigger model

```text
expiry_enabled: bool
usage_mode: disabled | threshold | zero
threshold_mb: int | null   # required iff usage_mode = threshold
```

| expiry_enabled | usage_mode | Effect |
|----------------|------------|--------|
| false | disabled | Invalid when `enabled=true` → **400** |
| true | disabled | Expiry only |
| false | threshold / zero | Usage only |
| true | threshold / zero | **expiry OR usage** |

### Must-lock semantics

#### 1. Reason precedence + no pending queue

In one beat evaluation, if both expiry and usage conditions are met:

1. Fire **once** with fire-reason `expiry`.
2. Do **not** enqueue, store, or “remember” a pending usage trigger.
3. The next beat re-evaluates usage from **fresh** usage state (subject to cooldown / freshness / rollout).

No in-memory or DB deferred-trigger queue.

#### 2. Trigger-config change resets cooldown

On successful policy upsert, if the persisted trigger-config tuple  
`(expiry_enabled, usage_mode, threshold_mb)` **changes**, set `cooldown_until = null`.

Does **not** reset solely because `enabled`, `package_id`, `renew_mode`, or `remaining_count` changed without a trigger-config change. If those change in the same upsert **and** the trigger tuple also changes, reset still applies.

#### 3. `usage_mode=zero` ≠ `threshold=0`

`zero` is a **semantically distinct** trigger from `threshold` with `threshold_mb=0`, even though both can result in 0 MB remaining.

| Mode | Condition (locked) | Fire reason / metric / idempotency reason |
|------|--------------------|-------------------------------------------|
| `threshold` | `remaining < threshold_mb` (same operator as v1) | `usage_threshold` |
| `zero` | `remaining <= 0` (exhausted; same as v1 `usage_zero`) | `usage_zero` |

Implementations **must not** collapse these into one branch. Reasons, Prometheus labels, and idempotency reason strings stay distinct.

#### 4. UI OR helper

When the user selects **both** expiry and remaining-data conditions, the UI **must** show helper copy equivalent to:

> Auto top-up will occur when **any** selected condition is met.

No AND wording.

---

## Explicitly out of v2 (separate epics)

- Always ask / notify-before-buy
- Auto Switch package ladder
- AND logic across conditions
- Arbitrary trigger lists / JSON rules / expression engines
- Multiple policies per eSIM
- Credit limit / overdraft (still requires future ADR amending ADR 010)
- Status / reason redesign beyond inherited v1 codes

---

## Domain model changes (relative to v1)

Keep `EsimAutoTopupPolicy`. Replace `trigger_mode` with:

| Field | Notes |
|-------|-------|
| `expiry_enabled` | bool |
| `usage_mode` | `disabled` \| `threshold` \| `zero` |
| `threshold_mb` | Required when `usage_mode=threshold`; null otherwise (CHECK) |

Unchanged: `account`, `esim`, `package_id`, `enabled`, `status`, `reason`, `renew_mode`, `remaining_count`, `cooldown_until`, last-trigger support fields, `version`.

Constraints: still **one policy per `esim`**.

---

## Beat evaluation (v2)

```text
Beat tick
  → Rollout / freshness / cooldown / minimum_age / status==active (as v1)
  → Select fire reason:
       if expiry_enabled and expired → reason=expiry
       elif usage_mode=threshold and remaining < threshold_mb → reason=usage_threshold
       elif usage_mode=zero and remaining <= 0 → reason=usage_zero
       else skip
  → TopupService.purchase (idempotent key including reason)
       → success: cooldown + events + metrics (incl. reason counter)
       → insufficient funds / package missing / timeout — same as v1
```

Unlimited packages/usage: usage modes rejected (as v1); expiry-only allowed.

---

## Idempotency keys (v2)

```text
auto-topup:{policy_id}:{reason}:{epoch}
```

| reason | epoch |
|--------|-------|
| `expiry` | `usage_expired_at` ISO (or documented unknown sentinel) |
| `usage_threshold` / `usage_zero` | `usage_synced_at` ISO |

Ledger debit keys remain `topup-debit:{topup.pk}`. Concurrent beats: `select_for_update` + unique idempotency → one top-up.

---

## Domain events

Inherited: `AutoTopupSucceeded`, `AutoTopupPolicyCreated`, `AutoTopupPolicyUpdated`, funds-pause path.

**Recommended (PR3, not a PR1 merge blocker):** `AutoTopupConfigurationChanged` with before/after `expiry_enabled`, `usage_mode`, `threshold_mb` — distinct from generic `AutoTopupPolicyUpdated` for product analytics.

---

## Metrics (Prometheus)

Inherited v1 counters/histograms, plus:

- `auto_topup_trigger_reason_total` labels: `reason=expiry|usage_threshold|usage_zero` (increment on successful auto top-up)

---

## API (breaking)

Path unchanged: `GET/PUT/DELETE /api/v1/me/esims/{id}/auto-topup/`

| Direction | Body / response |
|-----------|-----------------|
| Write / read | `expiry_enabled`, `usage_mode`, `threshold_mb` (no `trigger_mode`) |
| Unchanged | package, enabled, renew mode, count, status, reason, cooldown, version, If-Match → **409** |

OpenAPI regenerated in PR4; PR description includes a short breaking-change changelog. No `/api/v2` URL bump.

**PUT vs in-flight purchase:** if an auto-topup purchase / spend-in-progress is already active for that eSIM, PUT returns **409** with a retryable code/message.

---

## Web UI

On Available top-ups (`/me/esims/{id}`):

```text
When to auto top-up
☑ Current plan expires
☑ Remaining data
    (•) Below threshold → [MB]
    ( ) Reaches zero

(when both checked)
Auto top-up will occur when any selected condition is met.
```

- Client: require ≥1 condition when enabling
- Buy / shortfall UX unchanged

---

## Migration (PR2)

Forward map (**100% of rows**):

| v1 `trigger_mode` | v2 |
|-------------------|----|
| `expiry` | `expiry_enabled=true`, `usage_mode=disabled` |
| `usage_threshold` | `expiry_enabled=false`, `usage_mode=threshold` |
| `usage_zero` | `expiry_enabled=false`, `usage_mode=zero` |

Then drop `trigger_mode`. CHECKs for threshold. Prefer reversible migration while all rows remain 1:1 from v1.

PR4 DoD: serializer/OpenAPI contract tests assert **`trigger_mode` is absent**.

### Rollback (documented)

```text
Restore trigger_mode → reverse-map:
  (true, disabled) → expiry
  (false, threshold) → usage_threshold
  (false, zero) → usage_zero
  (true, threshold|zero) → NOT lossless
```

Combo policies (`expiry_enabled=true` and `usage_mode≠disabled`) cannot reverse to a single v1 `trigger_mode` without data loss. Prefer rolling back API/image **before** combo policies are common; if forced later, document mapping combo → `expiry` (usage leg discarded).

---

## DoD test matrix (v2 additions)

| Scenario | Expected |
|----------|----------|
| expiry + threshold both met same beat | one top-up, reason=expiry; no queued usage |
| next beat after expiry fire, still in cooldown | no usage top-up |
| after cooldown, usage still met | usage can fire from fresh state |
| PUT changes threshold while in cooldown | `cooldown_until` cleared |
| PUT changes only `package_id` | cooldown unchanged |
| `usage_mode=zero` vs threshold with mb=1 | distinct reasons/metrics/keys; never merged |
| enabled + no triggers | 400 |
| migration from each v1 mode | 100% mapped |
| OpenAPI / GET body | no `trigger_mode` |
| PUT during in-flight auto purchase | 409 |
| combo expiry+zero | OR works; helper visible in UI (PR5) |

Plus inherited v1 DoD (freshness, funds pause, package blocked, fixed_count, parallel beats, rollout, minimum_age).

---

## Implementation PR slice

| PR | Repo | Scope |
|----|------|--------|
| **PR1** | `roamkit-docs` | This design lock (docs only) |
| PR2 | `roamkit-api` | Schema + migration + admin field updates |
| PR3 | `roamkit-api` | Service + beat + must-lock behaviour + metrics + DoD tests |
| PR4 | `roamkit-api` | Me API + OpenAPI + contract tests + PUT 409 |
| PR5 | `roamkit-web` | Dual-condition UI + OR helper + e2e |

Never mix schema + UI in one PR. After this lock is merged: implement; do not redesign inside PR2–PR5.

---

## Architecture guardrails (non-negotiable)

1. Providers never touch money.
2. Only `CreditService` mutates balances/ledger (via existing top-up purchase).
3. Do not reuse `SubscriptionService` for usage triggers.
4. Idempotency + cooldown + freshness are layered; none replaces the others.
5. Never purchase on stale or failed usage refresh.
6. Provider timeout ≠ pause.
7. ADR 010 `balance >= 0` unchanged.
8. No pending usage queue after expiry fire.
9. Do not merge `zero` and `threshold` code paths.
