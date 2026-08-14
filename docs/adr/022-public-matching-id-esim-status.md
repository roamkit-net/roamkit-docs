# ADR 022: Public matching-id eSIM status

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Amended | **2026-08-14 (masked ICCID → full ICCID for display/copy)** |
| Deciders | Solo operator |
| Relates to | [ADR 021](./021-device-status-auth-iccid.md) (unchanged); [ADR 020](./020-organization-team-accounts.md) |

## Context

[ADR 021](./021-device-status-auth-iccid.md) locks managed-device status for BlackBerry UEM: serial or PR18 credential → UEM inventory → ICCID → unique non-archived `Esim` → read-only snapshot. That path is for `roamkit-bbuem-apk` (`net.roamkit.bbuem`) only.

A second product is needed: a **standalone consumer app** that shows the same kind of read-only status (usage, expiry, packages, coverage) **without** reading anything from the device, UEM, or a user login. The user supplies an Airalo / GSMA **Matching ID** (activation-code token), for example `TN2026060518450826EE68B1`.

`Esim.matching_id` already exists (provider fulfill). There is no public lookup, no unique constraint on the normalized value, and `/api/v1/me/esims/{id}/usage/` requires JWT + ownership.

This ADR locks a **new** public read-only capability path. It does **not** amend ADR 021.

### Amendment 2026-08-14 — full ICCID for display/copy

Original lock: `esim.iccid` on this path was **masked** (first 6 + last 4; shorter than 10 → `••••`). Never the full ICCID.

**Change:** `esim.iccid` is the **full** ICCID, for display and copy on RoamKit Status (same consumer display as the managed-device home card). Still **no** `packages.iccid`.

**Unchanged:**

- ICCID is **not** a resolve or auth field. Public lookup by ICCID stays **Rejected**.
- Request bodies, URLs, Sentry extras, and API/APK analytics or crash logs must **not** record a full ICCID (redact with `mask_iccid` or a `sha256` prefix).
- Full `matching_id`, LPA, QR, and install URLs stay forbidden on the response.

**Why:** a Matching ID is already a high-entropy read-only capability token. Hiding the ICCID after a successful resolve blocked support/copy UX without reducing who can read the snapshot.

This amend does **not** change [ADR 021](./021-device-status-auth-iccid.md). Implementation is a **separate API PR** after this docs PR merges. Do not implement API or APK in this docs PR.

## Decision

Introduce `POST /api/v1/public/esim/status/` authenticated only by a trimmed `matching_id` treated as a **read-only capability token**.

Ship a separate consumer APK later (not in this docs PR):

| Field | Value |
|-------|-------|
| GitHub repo | `roamkit-status-apk` |
| Local folder | `roamkit-status-apk` |
| Android `applicationId` | `net.roamkit.status` |
| App name | **RoamKit Status** |
| Relation to UEM | Fully standalone. Not a fork of `roamkit-bbuem-apk`. `roamkit-bbuem-apk` is not modified. |

```text
QR scan (local LPA parser)  or  manual matching_id (fallback)
        ↓
same validator → matching_id only
        ↓
POST /api/v1/public/esim/status/
        ↓
encrypted local active eSIM
```

v1 is **one** combined cache snapshot. Separate public `/packages/` and `/coverage/` siblings are **Rejected**. If they are needed later, open a new ADR.

### Delivery order (after this ADR is Accepted)

1. This ADR (docs only) — this PR.
2. API endpoint (separate PR, separate GO).
3. Create `roamkit-status-apk` and the Flutter APK (separate GO).

API and APK PRs must not start until this ADR is **Accepted**.

## Client contract (normative)

### v1 entry UX

```text
Matching ID
[ TN2026060518450826EE68B1 ] [ QR camera ]

                    [ Nastavi ]
```

- The user may type `matching_id` or tap the QR icon.
- A successful scan **fills only** the parsed `matching_id` into the same field. The user then taps **Nastavi**.
- Manual and scanned values use the **same** validator and the **same** public endpoint.
- If the camera is denied, manual entry still works.
- **Change eSIM** replaces the stored id (no server-side list).

### QR scan (v1 — local GSMA Activation Code parser)

QR is a v1 input method. It only extracts `matching_id` on device. The backend never receives QR, LPA, SM-DP+, or a confirmation code.

```text
LPA:1$consumer.e-sim.global$TN2026060518450826EE68B1
                                ↓
matching_id = TN2026060518450826EE68B1
```

Structured parse ([GSMA SGP.22](https://www.gsma.com/solutions-and-impact/technologies/esim/gsma_resources/sgp-23-v1-15/sgp-23_annexi_requirements_v1-15/) Activation Code: `LPA:1$<SM-DP+>$<activation-code>[$optional…]`). Do **not** take the last `$` segment.

1. `trim` the entire QR text.
2. Accept only if it starts with `LPA:1$`.
3. Split on `$`. Both required segments must be non-empty: `LPA:1$<non-empty SM-DP+>$<non-empty matching_id>`.
4. `matching_id` is the **third** segment (index 2). SM-DP+ is the second. Optional confirmation code and further segments are ignored.
5. The third segment uses the same validator as manual entry (`trim`, non-empty, `len ≤ 128`).

Reject examples: `LPA:1$$TN…` (empty SM-DP+) and `LPA:1$consumer.e-sim.global$` (empty matching id).

Guardrails:

- Persist only the parsed `matching_id` in **encrypted** storage. Never persist the full LPA, SM-DP+ address, or confirmation code.
- Process QR only locally. Do not send the image or raw string to the backend.
- Use the camera only while the scanner is open.
- The scanner must **not** install an eSIM and must **not** start Android LPA.
- Do not write QR / LPA content to logs, analytics, or Sentry.
- Invalid QR → **“Ovo nije valjani eSIM QR kod”**. Do not fill the input with the LPA string.

### Storage

`matching_id` is a secret on the device:

- Encrypted storage (for example EncryptedSharedPreferences / `flutter_secure_storage`).
- **Not** plain SharedPreferences.
- **Not** included in Android backup.
- **Not** present in analytics or crash logs.

### Standalone app — UEM strip (DoD)

`roamkit-status-apk` must not contain:

- managed configuration / `RestrictionsManager`
- `device_serial`, `device_external_id`, device credential
- binding / `binding_status`
- UEM provisioning
- UEM receivers or configuration listeners
- UEM names, copy, assets, or documentation
- UEM API calls (`/api/v1/device/*`)
- fallback to UEM mode
- UEM permissions or dependencies

Generic Flutter UI (usage ring, expiry, home-screen widget paint) may be copied only after all UEM assumptions and names are removed.

**APK PR test:** a repository-wide search must confirm there is no `UEM`, `bbuem`, `device_serial`, `device_external_id`, device/PR18 `credential`, or call to `/device/`.

**APK parser / storage tests:**

- valid LPA QR
- leading / trailing whitespace
- extra `$` segment (still the third segment)
- wrong prefix
- missing matching id, including `LPA:1$host$`
- empty SM-DP+ (`LPA:1$$TN…`)
- matching id too long
- ordinary URL or text QR
- full LPA is never stored
- successful parse fills the input with `matching_id` only
- no camera / permission denied: manual entry still works

## API contract (normative)

### Request

```http
POST /api/v1/public/esim/status/
{ "matching_id": "TN2026060518450826EE68B1" }
```

`matching_id` is the **only** request field and the only resolve key.

- Always `trim` the value.
- Do **not** accept `device_serial`, PR18 credential, `iccid`, `esim_id`, or a raw QR/LPA string as auth/resolve fields.
- **JWT:** do not reject a request merely because it has an `Authorization` header. JWT is unused and **must not** widen the response or privileges (same allow-list, same 404/200/400 semantics).

### HTTP errors

Separate a capability miss from a technically invalid HTTP request:

| Condition | Response |
|-----------|----------|
| empty, too long (`len > 128`), unknown, or ambiguous `matching_id` | `404 matching_id_not_found` |
| malformed JSON, or body that is not a JSON object | `400 invalid_request` |
| extra auth/resolve fields (`iccid`, `device_serial`, PR18 credential, raw LPA, …) | `400 invalid_request` — **do not ignore** |

`404 matching_id_not_found` means **only** that the Matching ID is empty, too long, or not uniquely found. A missing usage cache is **not** 404. A technically bad request is **not** 404.

### Resolve

```text
POST public/esim/status
      ↓
trim matching_id
      ↓
empty or len > 128          → 404 matching_id_not_found
      ↓
exactly one non-archived Esim with that matching_id
      ↓
0 or >1                     → 404 matching_id_not_found
exactly 1                   → 200 cache snapshot
```

- Never resolve with unordered `.first()`.
- “Non-archived” means `archived_at IS NULL`.
- Personal and team eSIMs are valid targets. Do not filter or rewrite `Esim.account`.
- One shared resolver builds status, packages, and coverage (one eSIM, one `checked_at`).
- One throttle namespace (for example `public_esim_status`, IP-based). Future siblings, if any, share that scope.

### Unique constraint and persist normalization (schema lock; implement in the API PR)

A unique index alone is not enough: `" TN123 "` would not match `"TN123"`.

Migration order:

1. Compute trimmed values for existing rows.
2. If trimmed non-empty values collide → **RAISE and abort the deploy**. Do not pick an eSIM.
3. If no collisions → **write** the trimmed values into `Esim.matching_id`.
4. Then add a partial unique constraint on non-empty trimmed `matching_id`.

- Multiple empty `matching_id` values remain allowed.
- v1 normalization is `trim` only (no case-fold). New writes always persist trimmed.
- Use a Postgres partial unique index, not a naive `unique=True` that would break empty rows.

### Cache only

This path must not call Airalo, `UsageService.get_usage`, or `PackageHistoryService`.

- Usage from `Esim.usage_*`. If there is no cache (`usage_synced_at` null / all usage fields empty) → **200** with `"usage": null`. The APK shows unavailable/stale, not “eSIM not found”.
- Coverage from `Order.coverage_snapshot`. No snapshot → `"coverage": null`.
- When `usage` is an object, include `synced_at` from `Esim.usage_synced_at`.

### Packages — do not lock reconstruction here

This ADR does **not** claim that `Order` + fulfilled `Topup` can reproduce a reliable package list.

The API PR must audit the models first:

- If the same display (statuses `active` / `not_active` / `queued` / `expired` / `finished`) can be built without a provider call and without inventing statuses → return `packages.results` + `active_package`.
- If it cannot → `"packages": null` and return only a reliable `plan` (title, allowance, validity). **Do not invent** package status.
- If a list is returned, omit unknown statuses; do not invent `queued`.

### Mutations (forbidden)

Knowing a Matching ID allows **view only**: status, usage, expiry, packages (when reliable), coverage, and whether auto top-up is on.

It must not authorize:

- purchase or top-up
- enabling, disabling, or changing auto top-up
- transfer / assign / unbind
- eSIM installation
- binding or credential changes
- ledger / balance mutations

### `auto_topup` (informational boolean only)

```json
{ "enabled": true }
```

```text
enabled = an AutoTopupPolicy exists for the resolved eSIM with status active
```

Defined on the domain model. No semantic dependency on the ADR 021 device snapshot.

Do **not** return: policy id, threshold, amount, selected package, cooldown, pause/block reason, purchase count, or payment data.

### Response schema (structure is normative; value types are not)

This ADR locks **field names, key presence, and null rules**. The JSON below is a **structure** example. It does **not** lock enums or whether usage amounts are strings, numbers, or bytes.

The API PR must first audit the existing status contract / Flutter parsers (`data_remaining`, `esim.status`, datetimes). Use those types. Do not accidentally lock `"1200 MB"` or `"in_use"` if the audit says otherwise.

Keys are always present. Missing data is JSON `null`, not an omitted key (except forbidden fields, which are never sent).

```json
{
  "esim": {
    "iccid": "89445012345678901234",
    "status": "in_use"
  },
  "usage": {
    "data_remaining": "1200 MB",
    "data_used": "800 MB",
    "expires_at": "2026-09-01T00:00:00Z",
    "synced_at": "2026-08-14T12:00:00Z"
  },
  "auto_topup": {
    "enabled": true
  },
  "plan": {
    "title": "Europe 5GB",
    "data_allowance": "5 GB",
    "validity_days": 30,
    "country_code": null,
    "coverage_type": "regional",
    "location_title": "Europe",
    "coverage_summary": {
      "available": true,
      "country_count": 32
    }
  },
  "packages": {
    "results": [],
    "active_package": null
  },
  "coverage": {
    "coverage_type": "regional",
    "coverage": [
      {
        "country_code": "HR",
        "country_name": "Croatia",
        "operators": ["Operator A"]
      }
    ]
  },
  "checked_at": "2026-08-14T12:05:00Z"
}
```

| Condition | Field |
|-----------|--------|
| No usage cache | `"usage": null` (200) |
| No reliable package history | `"packages": null` (200) |
| No coverage snapshot | `"coverage": null` (200) |
| No plan metadata | `"plan": null` (200) |
| Matching ID miss | 404 `matching_id_not_found` only |

- Full ICCID on `esim.iccid` only (display/copy). No `packages.iccid`.
- **Omit:** `device_external_id`, `binding_status`, `esim.id`.
- **Forbidden:** `lpa`, `qrcode*`, install URLs, full `matching_id` in the response, wholesale prices, balances, payment instruments, auto-topup fields beyond `enabled`.

### Logging / error tracking

Request bodies, URLs, Sentry extras, and API/APK analytics or crash logs must not record a full `matching_id`, full ICCID, or raw QR/LPA.

Allowed: redaction (`TN••••B1`) or a `sha256` prefix.

## Boundary with ADR 021

- ADR 021 is **not** amended. `POST /device/status|coverage|packages/` stays UEM-only.
- Folding packages and coverage into **this** public snapshot is intentional. ADR 021 still forbids folding them into the device status/coverage endpoints.
- This path must not accept device serial or PR18 credentials.

## Non-goals (this docs PR)

- API implementation, migration, OpenAPI, throttle rate numbers
- Auditing usage/enum types (API PR)
- Auditing `Order` + Topup → packages (API PR)
- Creating `roamkit-status-apk` / `net.roamkit.status`
- A page in `roamkit-web`
- Live Airalo refresh
- Any change to `roamkit-bbuem-apk` or to [ADR 021](./021-device-status-auth-iccid.md)

## Rejected

| Option | Status |
|--------|--------|
| Extending `POST /device/status/` with `matching_id` | **Rejected** (would amend ADR 021) |
| Public lookup by ICCID | **Rejected** (ADR 021: ICCID is never a credential) |
| Separate v1 public `/packages/` and `/coverage/` | **Rejected** |
| JWT required, or JWT widening the snapshot | **Rejected** |
| Rejecting a request only because `Authorization` is present | **Rejected** |
| Live provider calls on this path | **Rejected** |
| 404 when usage cache is missing | **Rejected** |
| Inventing package statuses without a model audit | **Rejected** |
| Sending QR / LPA to the backend | **Rejected** |
| Installing eSIM / starting Android LPA from the scanner | **Rejected** |
| UEM config, serial, device credential, or login in RoamKit Status | **Rejected** |
| Mutations authorized by Matching ID | **Rejected** |
| Plain SharedPreferences / Android backup of `matching_id` | **Rejected** |

**QR scan is part of the v1 APK; it only extracts `matching_id` locally.**

## Consequences

### Positive

- Consumer devices can view cached usage and expiry without UEM or login.
- One request returns a consistent cache snapshot.
- Matching ID stays a capability token: read-only, throttled, redacted, encrypted at rest on the device.
- ADR 021 and `roamkit-bbuem-apk` stay isolated.

### Negative / accepted tradeoff

- Anyone who knows a Matching ID can read the allow-listed snapshot, including the full ICCID (2026-08-14 amend). Accepted because the path is mutation-free and the token is high-entropy (Airalo activation code).
- Package history may be `null` until the API PR audit succeeds.
- Usage may be stale; `usage.synced_at` (or `usage: null`) makes that visible.

### Follow-up (not this PR)

- API PR: normalize + unique `matching_id`, public endpoint, tests, type audit, package audit (done).
- API PR (this amend): return full `esim.iccid`; keep log redaction; allow-list regression.
- APK PR: `roamkit-status-apk`, UX above, encrypted storage, UEM-strip grep, parser tests.

## Stop rule

While this ADR is **Proposed**:

- Do not start the API or APK PRs.

Once **Accepted**:

- Implementation must follow this document.
- Do not silently change architecture during implementation.
- Do not amend [ADR 021](./021-device-status-auth-iccid.md) to add Matching ID to the device family.
- Do not authorize mutations from Matching ID.
- Do not call the provider on this path.
- Do not return a full LPA or Matching ID on this path.
- Do not **log** a full ICCID, LPA, or Matching ID.
- Do not treat a missing cache as `matching_id_not_found`.
- Do not invent package statuses.
- Do not persist or log raw QR / LPA.

Further scope (new endpoints, login, mutations, UEM leftovers) requires a new ADR.

## Related

- [ADR 021](./021-device-status-auth-iccid.md) — managed-device status (unchanged)
- [ADR 020](./020-organization-team-accounts.md) — Account ownership (unchanged)
- [ADR 014](./014-esim-lifecycle-install-telemetry.md) — eSIM lifecycle statuses
