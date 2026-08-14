# ADR 021: Device status auth (ICCID lookup vs credentials)

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-09 |
| Amended | 2026-08-09 (serial bootstrap); 2026-08-09 (serial-only v1 status); 2026-08-10 (ICCID lookup ≠ ownership); 2026-08-10 (no DeviceBinding gate for serial status); **2026-08-14 (device packages + ICCID display + local paid_usd)** |
| Deciders | Solo operator |
| Relates to | [ADR 020](./020-organization-team-accounts.md) |

## Context

[ADR 020](./020-organization-team-accounts.md) reserved a device / UEM extension surface:

- `DeviceBinding` attaches to Account-owned eSIMs via a RoamKit-issued `device_external_id`
- A managed device profile carries that id; the client calls a status API
- No BlackBerry UEM admin sync in v1 of ADR 020

That shape shipped as **PR18**: public `POST /api/v1/device/status/` authenticates with `device_external_id` + opaque per-binding credential (hash at rest; plaintext only at issue/rotate). `device_external_id` is a lookup key only — never sufficient authorization.

PR18 is **confirmed end-to-end** on a real BlackBerry-managed Pixel 6a (UEM managed config → APK → status API → active `DeviceBinding` → usage snapshot).

Operators need **one** UEM App Configuration per organization — not a new managed-config secret per eSIM or per SIM swap. Local ICCID on the APK was tested and **failed**. Read-only UEM REST can supply ICCID after the physical device is identified via serial.

Product intent for managed-device v1 is a **read-only status / usage / packages view** (low-sensitivity). Mutations (purchase, top-up, auto-topup changes, inventory moves, binding changes) must **never** be authorized by this path.

Hard rules:

1. **ICCID must never become a credential.** No unauthenticated `GET /device/status/{iccid}`. Device request bodies must **not** accept a client-supplied ICCID (alone or with serial/PR18). The server resolves ICCID from the authenticated UEM device.
2. **Lookup and mutation stay separate.** Serial-based status/coverage/packages are read-only only.
3. **`Esim.account` / `Order.account` remain the purchase ownership boundary.** MDM/UEM enrollment must **never** transfer or rewrite eSIM ownership. Inventory ownership changes only via ADR 020 explicit Account → Account transfer.
4. **UEM `device.guid` alone is never sufficient authorization.**
5. **Device serial is an identifier, not a secret.** It must not authorize mutations. For v1 read-only status/coverage/packages it is the public device lookup key: **serial → current UEM inventory → ICCID → unique non-archived Esim**. An active `DeviceBinding` is **not** required on this path.
6. **No free-form serial entry in the APK** — serial comes from UEM App Configuration `%SerialNumber%` only.
7. **Read-only status ICCID lookup is not scoped to any organization Account.** Personal and team eSIMs are both valid resolve targets.
8. **Serial status/coverage/packages MUST NOT auto-create `DeviceBinding`.** Binding rows are enrollment/admin/PR18 concerns, not a side effect of read.

## Spike results (historical + amend proof)

### Local ICCID (negative proof)

Validated on BlackBerry UEM Cloud tenant device (Pixel 6a, Android 16, work profile):

| Source | Result |
|--------|--------|
| BlackBerry UEM device report / REST | ICCID **visible** |
| APK local read (`SubscriptionInfo.getIccId()`) | **ICCID not readable** |

**Local-ICCID hybrid is rejected** and must not be reopened without new positive proof on real BlackBerry-managed devices.

Non-Dynamics Android **app config create/update is not supported** via UEM REST — operators cannot automate per-device App Configuration writes for this package.

### UEM App Configuration variable expansion (positive proof)

Validated on tenant `S31564560` with debug package `net.roamkit.mdmspike` (Pixel 6a):

| Key / value in UEM App Config | Delivered to APK |
|-------------------------------|------------------|
| `spike.literal_marker` = `SPIKE_OK` | `SPIKE_OK` (delivery works) |
| `spike.serial_number` = `%SerialNumber%` | `36281JEGR04531` (**expanded**) |
| `spike.device_guid` = `%DeviceGUID%` | literal `%DeviceGUID%` (not a recognized variable) |
| `spike.device_uid` = `%DeviceUID%` | literal `%DeviceUID%` |
| `spike.imei` = `%IMEI%` | literal `%IMEI%` |
| `spike.iccid` = `%ICCID%` | literal `%ICCID%` |
| `spike.wifi_mac` = `%WiFiMAC%` | literal `%WiFiMAC%` |

**Conclusion:** BlackBerry UEM **does** perform per-device substitution into Android managed App Configuration for at least `%SerialNumber%`. That is sufficient for one shared App Configuration that still yields a per-phone identity on device.

### UEM REST identity mapping (read-only)

Tenant `S31564560` (`GET /api/v1/devices`, `MDMBWS.All`):

| Finding | Detail |
|---------|--------|
| Stable APK bootstrap identity | UEM **`serialNumber`** via App Config `%SerialNumber%` |
| Current UEM record identity | UEM **`device.guid`** (may change after wipe / re-enroll) |
| ICCID as device id | **Rejected** (duplicate ICCID across two devices observed) |
| Dual-SIM | Top-level `iccid` ∈ `sims[]`, but **not always** `sims[0]` |
| REST detail by guid | `GET /devices/{guid}` returned **404** on this tenant |
| Direct `query=serialNumber=…` | **HTTP 400** — `Unknown query field: serialNumber` |
| Direct `query=imei=…` / `query=udid=…` | Works on this tenant (optional optimization later) |
| List + match `serialNumber` | **Validated** — finds the Pixel among listed devices |

Validated UEM read path for serial status (current amend):

```text
GET /api/v1/devices
      ↓
match device.serialNumber == request device_serial (exactly one)
      ↓
use that record's authoritative ICCID
```

Historical note: an earlier C″ draft matched UEM serial against `DeviceBinding.uem_serial_number` as a gate. That binding gate is **removed** for read-only serial status/coverage by this amend.

### GUID vs serial lifecycle (proof)

Same Pixel after re-activation:

| Field | Before | After |
|-------|--------|-------|
| `serialNumber` | `36281JEGR04531` | `36281JEGR04531` (stable) |
| `device.guid` | `bc473029-…` | `fb3de589-…` (changed) |
| ICCID | RoamKit eSIM ICCID (personal or team) | same ICCID after inventory refresh |

```text
device serial   = physical phone identity (stable across re-enroll)
device.guid     = current UEM inventory record (may change)
ICCID           = current SIM state / lookup only
```

Empty telephony inventory means **UEM inventory unavailable/stale**, not “no eSIM”.

## Decision (Accepted — amended: no DeviceBinding gate for serial status)

**Option C″ — `%SerialNumber%` + UEM-sourced ICCID + unique Esim (read-only status/coverage)** is the normative managed-device **v1 status** target.

**Option A — PR18** remains the shipped contract and **required fallback** through migration (`device_external_id` + credential → binding → then same UEM/ICCID rules where applicable).

**Option B / local hybrid** remain **rejected**.

**Pairing-only guid enrollment** remains **superseded**.

**Option C′ fleet credential on the status path** (`fleet_external_id` + `fleet_credential` as status auth) is **superseded for v1 status**. Fleet credential schema/services may remain in the codebase as optional future hardening, but they are **not** part of the Accepted v1 device status flow or App Configuration.

### Accepted security tradeoff (explicit)

For v1 read-only status/usage/coverage/packages:

- Knowing a **UEM-known** device serial allows reading that device’s status/usage/coverage snapshot and applied package history.
- That is accepted because the path is **mutation-free**. Allowed fields are listed under **Device response allow-list**.
- Serial **without** an active `DeviceBinding` is **accepted** on this path (no `binding_not_found` for serial status/coverage/packages).
- Serial must **never** authorize purchase, top-up, auto-topup changes, eSIM transfer, binding changes, or return of account balances, payment instruments, or Airalo wholesale prices.
- Do **not** auto-create `DeviceBinding` to “satisfy” serial status.

### Role separation (normative)

```text
device_serial         = identify physical phone (UEM %SerialNumber%; not a secret)
UEM inventory         = authoritative current device + ICCID for serial status/coverage
ICCID                 = find current eSIM (lookup only; never ownership rewrite)
Esim.account          = purchase ownership (personal or team); unchanged by MDM paths
DeviceBinding         = PR18 credential enrollment / admin mapping only — NOT a serial status gate
```

### Authorization / resolution boundary (v1 serial status/coverage)

```text
device_serial from request
AND
UEM serial match count == 1 → device record
AND
usable authoritative ICCID from that record
AND
exactly one non-archived Esim with that ICCID in RoamKit
```

**Forbidden for serial status/coverage:**

- requiring an active `DeviceBinding` for the serial
- requiring `Esim.account == DeviceBinding.organization.account`
- auto-creating or updating `DeviceBinding` on the read path
- using `DeviceBinding` serial/GUID/cache fields as a **fallback resolver** when UEM fails

No fleet credential check on this path. GUID alone is never enough. ICCID alone is never enough. MDM bind / rotate / status must never mutate `Esim.account` or `Order.account`.

### UEM App Configuration (ops lock — v1 status)

**One** App Configuration per organization for `net.roamkit.bbuem` (identical template; serial expands per device):

| Key | Role |
|-----|------|
| `roamkit.device_serial` | Must be UEM variable **`%SerialNumber%`** |

`roamkit.fleet_external_id` and `roamkit.fleet_credential` are **not** required for v1 status and must not be treated as normative App Config for this flow.

No new managed-config values on eSIM swap. No free-form / user-typed serial entry in the APK.

### DeviceBinding identity fields (enrollment / PR18 only)

| Field | Role |
|-------|------|
| `uem_serial_number` | Optional ops/admin map for enrollment surfaces — **not** required for serial status |
| `uem_device_guid` | Optional UEM record correlation / cache for enrollment — **not** a serial status fallback |

`DeviceBinding` remains for PR18 (`device_external_id` + credential), admin enrollment, and future write/enrollment surfaces. First schema work keeps `DeviceBinding.esim` non-null (PR18 / legacy). Nullable cleanup remains a later PR.

### Status / coverage resolution (serial path — normative v1)

Applies identically to `POST /api/v1/device/status/`, `POST /api/v1/device/coverage/`, and `POST /api/v1/device/packages/` (packages then loads history; it does not change this resolve):

```text
POST /api/v1/device/status/  (or .../coverage/)
  { "device_serial": "..." }
      ↓
UEM GET /devices + match serialNumber
      ↓
exactly one device → authoritative ICCID
      ↓
Esim by ICCID (global RoamKit domain; uniqueness guard)
      ↓
read-only snapshot
  device_external_id = null
```

Serial status/coverage resolution MUST use the **current UEM inventory** as the authoritative source. `DeviceBinding` serial/GUID/cache fields MUST NOT be used as a fallback resolver. The path MUST NOT create, update, or delete `DeviceBinding` rows.

Personal-account eSIMs are valid resolve targets; MDM enrollment does **not** require moving inventory onto an organization Account.

### Device packages sibling (normative — 2026-08-14 amend)

`POST /api/v1/device/packages/` is a third **read-only** sibling of status and coverage. Same auth shapes (serial **or** PR18, never mixed), same UEM → ICCID → unique non-archived `Esim` resolve, same failure codes, no mutations, no `DeviceBinding` side effects.

```text
POST /api/v1/device/packages/
  { "device_serial": "..." }          ← preferred
  or { "device_external_id", "credential" }
      ↓
same C″ / PR18 resolve as status
      ↓
server-chosen Esim.iccid (never from the request body)
      ↓
read-only applied package history + local paid_usd match
```

**Request:** the body must **not** accept `iccid` (or any client-chosen SIM identifier). The APK never sends an arbitrary ICCID. The endpoint selects ICCID only from the authenticated UEM device.

**Provider call:** allowed **only** on this packages endpoint (Airalo `GET /v2/sims/{iccid}/packages` via `PackageHistoryService`). Do **not** fold package history into `POST /device/status/` or `POST /device/coverage/`. Status and coverage stay **cache / purchase-snapshot only**.

**`paid_usd`:** local `Order.retail_price_usd` / fulfilled `Topup.amount` only, with the same fail-closed matching as the user `GET /me/esims/{id}/packages/` contract (identity first; unambiguous `package_external_id` only; else `null`). Never Airalo wholesale `price` / `net_price`.

### Device response allow-list (normative — 2026-08-14 amend)

Device responses on this family **may** return:

- the **full** resolved `Esim.iccid` (APK home may display and copy it)
- **applied package history** on `POST /api/v1/device/packages/` only
- local retail **`paid_usd` + `currency`** from Order/Topup matching

Device responses **must never** return:

- Airalo wholesale `price` / `net_price`
- account balances
- payment instruments
- other high-sensitivity account data beyond this allow-list and the existing status/coverage snapshot fields

ICCID remains **not a credential**. It must not appear on home-screen widgets or in logs. A client-supplied ICCID is never authorization and is never a request field.

### Response contract — `device_external_id` (normative)

The field remains present on both shapes so the JSON contract stays stable for clients:

| Path | `device_external_id` |
|------|----------------------|
| PR18 (`device_external_id` + `credential`) | binding’s external id string |
| Serial (`device_serial`) | **`null`** |

For serial status/coverage resolution, `device_external_id` MUST be `null`; the API MUST NOT synthesize or infer a DeviceBinding external id. The field remains present (not omitted) to preserve the existing response contract. Serial path identity is **serial + resolved esim** (plus usage/plan/coverage fields).

### ICCID uniqueness (normative)

`Esim.iccid` is already **globally unique** at the schema layer. Read-only status/coverage resolve must still apply an application uniqueness guard and **must not** use unordered `.first()`:

| Non-archived `Esim` rows with ICCID | Result |
|-------------------------------------|--------|
| exactly 1 | resolve that eSIM |
| 0 | `iccid_not_found` |
| >1 | `iccid_ambiguous` (fail closed; defense in depth if uniqueness is ever relaxed) |

“Non-archived” means `archived_at IS NULL`. An archived-only row does not satisfy status resolve (`iccid_not_found`). Ambiguity must never return an arbitrary eSIM.

PR18 body shape (`device_external_id` + `credential`) remains supported as fallback on the same endpoints. When PR18 resolves via UEM ICCID, the same global uniqueness rule applies. Serial shape and PR18 shape must not be mixed in one request.

### Read-only hard stop (normative)

The serial status/coverage/packages path (and PR18 equivalents) must remain **strictly read-only**. They must not:

- purchase or top-up
- create/change/disable auto-topup
- transfer / assign / unbind eSIMs
- change `Esim.account` / `Order.account` (ownership)
- create/rotate/revoke bindings or credentials (including as a “helpful” side effect of status)
- return account balances, payment instruments, or Airalo wholesale prices (see **Device response allow-list**)

`create_device_binding` / credential rotate are enrollment operations: they may create or update `DeviceBinding` rows, but they **must not** rewrite eSIM ownership as a side effect.

### Dual-SIM rule (normative)

- Use **top-level `iccid`** when present (on this tenant it is always one of `sims[].iccid`).
- Do **not** assume `iccid == sims[0].iccid`.
- If top-level missing and/or `sims=[]` → `uem_inventory_unavailable` (fail closed; no inventory mutation).

### GUID / serial lifecycle (normative)

- Serial status/coverage always re-resolves via **current** UEM list + serial match. No pairing. No silent remap by ICCID. No binding-cache rescue.
- Enrollment/admin code may still refresh a binding’s cached `uem_device_guid` after a unique serial match; that is **not** part of the serial status/coverage read path and must not be required for success.
- ICCID changes (SIM swap) do **not** require App Configuration changes.

### Distinct device API failure codes (normative)

Do **not** collapse these into one generic 404. Do **not** collapse zero vs multiple UEM serial matches:

| Condition | Code |
|-----------|------|
| UEM serial match = 0 | `device_not_found` |
| UEM serial match > 1 | `device_ambiguous` |
| Exactly one UEM device, but no usable ICCID / empty telephony | `uem_inventory_unavailable` |
| UEM ICCID present, no non-archived RoamKit `Esim` with that ICCID | `iccid_not_found` |
| UEM ICCID present, more than one non-archived RoamKit `Esim` with that ICCID | `iccid_ambiguous` |
| PR18 / enrollment: no active binding for credential lookup (not used on serial status/coverage) | `binding_not_found` |

On all families: **no** create/transfer/unbind/ownership/provider **mutations**. Status and coverage must not call the provider. Packages may perform a **read-only** provider history fetch. Stale `DeviceBinding` rows MUST NOT override a UEM failure (regression: binding exists + UEM miss → UEM failure code, not old eSIM).

### REST constraint (validated tenant)

On tenant `S31564560` (`p07003.cp1.uem.blackberry.com`):

> - `GET /devices/{guid}` is **not** available (**404**).
> - `query=serialNumber=…` is **not** supported (**400** Unknown query field).
> - Validated path: **`GET /devices` + match `serialNumber`**.
> - Do not assume detail-by-guid or serial query filters unless proven later on this tenant.

### Mapping keys

```text
APK / App Config     : roamkit.device_serial = %SerialNumber%
Authoritative device : current UEM inventory (serial match)
Current SIM lookup   : UEM ICCID → unique non-archived Esim.iccid (any Account)
DeviceBinding        : PR18 / enrollment / admin only (not serial status gate or fallback)
```

ICCID must **not** be treated as device identity for auth. Organization on `DeviceBinding` is an enrollment context, not an eSIM owner filter for serial status.

### Rejected / superseded options

| Option | Status |
|--------|--------|
| A — PR18 only as long-term exclusive model | Superseded as **target**; retained as **fallback** |
| B — Pure fleet + local ICCID | **Rejected** |
| Local hybrid (fleet + local ICCID + binding gate) | **Rejected** |
| C — Fleet + UEM ICCID + pairing-only guid | **Superseded** |
| C′ — Fleet credential + `%SerialNumber%` on status | **Superseded for v1 status** by C″ |
| C″ — Serial + UEM ICCID + unique Esim (read-only; no binding gate) | **Accepted** (this amend) |
| Serial with **no** DeviceBinding gate | **Accepted** for read-only status/coverage/packages |
| Device packages sibling (`POST /device/packages/`) | **Accepted** (2026-08-14 amend) |
| Client-supplied ICCID on device request bodies | **Rejected** |
| Wholesale Airalo price on device responses | **Rejected** |
| Folding package history into status/coverage | **Rejected** |
| DeviceBinding cache as fallback when UEM fails | **Rejected** |
| Auto-create DeviceBinding on serial status/coverage/packages | **Rejected** |
| Serial authorizing mutations | **Rejected** |
| MDM bind requiring / performing `Esim.account` transfer onto org Account | **Rejected** |
| Status resolve requiring `Esim.account == organization.account` | **Rejected** (2026-08-10 amend) |

### Migration / compatibility

- PR18 `device_external_id + credential` remains supported through migration and as rollback.
- Existing `OrganizationFleetCredential` / fleet status auth shape (if already shipped) is **non-normative for v1** after this amend; do not require fleet keys in UEM App Config for status. Removal or deprecation of fleet status auth is a follow-up implementation PR, not this docs PR.
- Deprecate PR18 device credentials in UEM only after serial status is validated on fleet devices.
- Implementation of this amend is a **separate API PR** (docs ≠ API ≠ APK).

### Non-goals (this docs amend)

- Implementing API / APK / web changes in this docs PR
- Making `DeviceBinding.esim` nullable
- BlackBerry UEM write sync
- Free-form serial entry in the APK
- Pairing as v1 enrollment
- Resurrecting local ICCID without a new spike proof
- Using serial-only status for any mutating API
- Auto-creating DeviceBinding from status/coverage/packages
- Implementing `POST /device/packages/` or APK UI in this docs PR
- Accepting a client ICCID on any device request body

## Consequences

### Accepted Option C″ (this amend)

- Normative v1 App Config for status: only `roamkit.device_serial=%SerialNumber%`.
- Normative serial resolve: current UEM inventory → ICCID → unique non-archived `Esim` (personal or team) → read-only snapshot.
- No `DeviceBinding` gate, no binding auto-create, no binding-cache fallback on serial status/coverage.
- Serial responses set `device_external_id` to **`null`** (field present).
- Explicit tradeoff: UEM-known serial ⇒ read status/usage/coverage/packages (full ICCID, local `paid_usd`); never mutations, ownership transfer, or wholesale prices.
- `POST /device/packages/` is the only device path that may call the provider (read-only history). Status/coverage stay cache-only.
- APK may show and copy the resolved ICCID on the in-app home screen; widgets and logs must not.
- Pairing out of v1. Fleet credentials out of v1 status App Config / status auth.
- PR18 remains fallback until explicitly deprecated after validation.
- Implementation continues in small PRs (docs ≠ API ≠ APK).

### Rejected paths

- Local ICCID / hybrid remain closed.
- ICCID-as-auth and guid-alone-as-auth remain forbidden.
- Binding-gate and binding-fallback for serial status remain forbidden.
- Serial as mutation auth remains forbidden.
- Silent / MDM-driven `Esim.account` transfer remains forbidden.

## Stop rule

Now that this ADR is **Accepted** (as amended — device packages + ICCID display + local `paid_usd`):

- Implementation **must** follow Option C″ for v1 managed-device status/coverage/packages (or open a new ADR discussion first).
- Do **not** require `fleet_*` App Config keys for v1 status.
- Do **not** require an active `DeviceBinding` for serial status/coverage/packages.
- Do **not** auto-create or update `DeviceBinding` on serial status/coverage/packages.
- Do **not** use `DeviceBinding` serial/GUID/cache as a fallback when UEM resolve fails.
- Do **not** authorize any mutation from serial (or from this status/coverage/packages endpoint family).
- Do **not** require `Esim.account == DeviceBinding.organization.account` for read-only status/coverage/packages resolve.
- Do **not** transfer or rewrite `Esim.account` / `Order.account` from bind, rotate, serial, or UEM status paths.
- Do **not** resolve ICCID with unordered `.first()` when multiple non-archived rows match — fail closed (`iccid_ambiguous`).
- Do **not** collapse UEM serial match 0 and >1 into one code — use `device_not_found` vs `device_ambiguous`.
- Do **not** omit `device_external_id` on serial success — return JSON `null`.
- Do **not** treat UEM ICCID or UEM `guid` alone as sufficient authorization.
- Do **not** treat `iccid=null` / `sims=[]` as proof of no eSIM or as a trigger to mutate inventory.
- Do **not** implement local-ICCID status paths without a new ADR.
- Do **not** reintroduce pairing or fleet-credential status auth as required v1 without a new ADR discussion.
- Do **not** assume `query=serialNumber` on this tenant without re-validation.
- Do **not** accept a client-supplied ICCID on `POST /device/status|coverage|packages/`.
- Do **not** fold package history into status/coverage.
- Do **not** return Airalo wholesale `price` / `net_price` on any device response.
- Do **not** start the API or APK packages PRs until this amend is Accepted (this docs PR).

Changing this Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
