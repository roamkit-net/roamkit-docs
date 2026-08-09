# ADR 021: Device status auth (ICCID lookup vs credentials)

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-09 |
| Amended | 2026-08-09 (serial bootstrap); **2026-08-09 (serial-only v1 status)** |
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

Product intent for managed-device v1 is a **read-only status / usage widget** (low-sensitivity). Mutations (purchase, top-up, auto-topup changes, inventory moves, binding changes) must **never** be authorized by this path.

Hard rules:

1. **ICCID must never become a credential.** No unauthenticated `GET /device/status/{iccid}` (or body with ICCID alone).
2. **Lookup and mutation stay separate.** Serial-based status is read-only only.
3. **`Esim.account` remains the ownership boundary** for resolved inventory.
4. **UEM `device.guid` alone is never sufficient authorization.**
5. **Device serial is an identifier, not a secret.** It must not authorize mutations. For v1 status it is accepted as the public device lookup key **only together with** an active `DeviceBinding` for that serial.
6. **No free-form serial entry in the APK** — serial comes from UEM App Configuration `%SerialNumber%` only.

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

Validated read path:

```text
GET /api/v1/devices
      ↓
match device.serialNumber == DeviceBinding.uem_serial_number
      ↓
use that record's current device.guid + authoritative ICCID
```

### GUID vs serial lifecycle (proof)

Same Pixel after re-activation:

| Field | Before | After |
|-------|--------|-------|
| `serialNumber` | `36281JEGR04531` | `36281JEGR04531` (stable) |
| `device.guid` | `bc473029-…` | `fb3de589-…` (changed) |
| ICCID | team eSIM ICCID | same ICCID after inventory refresh |

```text
device serial   = physical phone identity (stable across re-enroll)
device.guid     = current UEM inventory record (may change)
ICCID           = current SIM state / lookup only
```

Empty telephony inventory means **UEM inventory unavailable/stale**, not “no eSIM”.

## Decision (Accepted — amended for serial-only v1 status)

**Option C″ — `%SerialNumber%` + active DeviceBinding + UEM-sourced ICCID (read-only status)** is the normative managed-device **v1 status** target.

**Option A — PR18** remains the shipped contract and **required fallback** through migration.

**Option B / local hybrid** remain **rejected**.

**Pairing-only guid enrollment** remains **superseded**.

**Option C′ fleet credential on the status path** (`fleet_external_id` + `fleet_credential` as status auth) is **superseded for v1 status**. Fleet credential schema/services may remain in the codebase as optional future hardening, but they are **not** part of the Accepted v1 device status flow or App Configuration.

### Accepted security tradeoff (explicit)

For v1 read-only status/usage:

- Knowing an **enrolled** device serial allows reading that device’s status/usage snapshot.
- That is accepted because the payload is low-sensitivity and the path is **mutation-free**.
- Serial **without** an active `DeviceBinding` must fail closed (`binding_not_found`).
- Serial must **never** authorize purchase, top-up, auto-topup changes, eSIM transfer, binding changes, or return of account secrets / payment data.

### Role separation (normative)

```text
device_serial         = identify physical phone (UEM %SerialNumber%; not a secret)
active DeviceBinding  = enrollment / org gate for that serial
UEM device.guid       = current UEM inventory record (correlation / cache)
ICCID                 = find current eSIM (lookup only)
```

### Authorization / resolution boundary (v1 status)

```text
device_serial from request
AND
active DeviceBinding(uem_serial_number == device_serial)
AND
UEM unique serial match → ICCID
AND
Esim.account == DeviceBinding.organization.account
```

No fleet credential check on this path. GUID alone is never enough. ICCID alone is never enough. Serial without active binding is never enough.

### UEM App Configuration (ops lock — v1 status)

**One** App Configuration per organization for `net.roamkit.bbuem` (identical template; serial expands per device):

| Key | Role |
|-----|------|
| `roamkit.device_serial` | Must be UEM variable **`%SerialNumber%`** |

`roamkit.fleet_external_id` and `roamkit.fleet_credential` are **not** required for v1 status and must not be treated as normative App Config for this flow.

No new managed-config values on eSIM swap. No free-form / user-typed serial entry in the APK.

### DeviceBinding identity fields (normative)

| Field | Role |
|-------|------|
| `uem_serial_number` | **Stable** map key for the physical device |
| `uem_device_guid` | **Current** UEM record correlation / cache; refreshable after unique serial match |

Operator creates / maintains an active `DeviceBinding` with `uem_serial_number` (once per physical phone, not per eSIM). First schema work keeps `DeviceBinding.esim` non-null (PR18 / legacy). Nullable cleanup remains a later PR.

### Status resolution (serial path — normative v1)

```text
POST /api/v1/device/status/
  { "device_serial": "..." }
      ↓
require active DeviceBinding(uem_serial_number == device_serial)
      ↓
UEM GET /devices + match serialNumber (exactly one)
      ↓
current device.guid (+ refresh cache) + authoritative ICCID
      ↓
Esim on DeviceBinding.organization.account
      ↓
read-only status snapshot
```

This path **ignores** `DeviceBinding.esim` for resolving the current eSIM (ICCID comes from UEM).

PR18 body shape (`device_external_id` + `credential`) remains supported as fallback on the same endpoint (or as already shipped). Serial shape and PR18 shape must not be mixed in one request.

### Read-only hard stop (normative)

The serial status path (and PR18 status path) must remain **strictly read-only**. They must not:

- purchase or top-up
- create/change/disable auto-topup
- transfer / assign / unbind eSIMs
- create/rotate/revoke bindings or credentials
- return account balances, payment instruments, or other high-sensitivity account data beyond the existing status snapshot fields

### Dual-SIM rule (normative)

- Use **top-level `iccid`** when present (on this tenant it is always one of `sims[].iccid`).
- Do **not** assume `iccid == sims[0].iccid`.
- If top-level missing and/or `sims=[]` → `uem_inventory_unavailable` (fail closed; no inventory mutation).

### GUID / serial lifecycle (normative)

- If UEM `device.guid` changes after wipe/re-enroll but **serial is unchanged**: RoamKit **re-discovers** via serial match and may update cached `uem_device_guid`. No pairing. No silent remap by ICCID.
- If serial cannot be matched uniquely in UEM inventory → fail closed (`uem_inventory_unavailable`); do not invent a device.
- Serial with no active binding → `binding_not_found`.
- ICCID changes (SIM swap) do **not** require App Configuration changes.

### Distinct device API failure codes (normative)

Do **not** collapse these into one generic 404:

| Condition | Code |
|-----------|------|
| No active binding for serial; unbound/replaced | `binding_not_found` |
| UEM ICCID present, no team-Account `Esim` for the binding’s org | `iccid_not_found` |
| UEM serial match not exactly one; or telephony empty/stale | `uem_inventory_unavailable` |

On all three families: **no** create/transfer/unbind/provider side effects.

### REST constraint (validated tenant)

On tenant `S31564560` (`p07003.cp1.uem.blackberry.com`):

> - `GET /devices/{guid}` is **not** available (**404**).
> - `query=serialNumber=…` is **not** supported (**400** Unknown query field).
> - Validated path: **`GET /devices` + match `serialNumber`** (exactly one).
> - Do not assume detail-by-guid or serial query filters unless proven later on this tenant.

### Mapping keys

```text
APK / App Config     : roamkit.device_serial = %SerialNumber%
Stored stable map key: DeviceBinding.uem_serial_number
Stored UEM cache     : DeviceBinding.uem_device_guid (refreshable)
Current SIM lookup   : UEM ICCID → Esim.iccid on binding organization.account
```

ICCID must **not** be the DeviceBinding ↔ UEM device identity.

### Rejected / superseded options

| Option | Status |
|--------|--------|
| A — PR18 only as long-term exclusive model | Superseded as **target**; retained as **fallback** |
| B — Pure fleet + local ICCID | **Rejected** |
| Local hybrid (fleet + local ICCID + binding gate) | **Rejected** |
| C — Fleet + UEM ICCID + pairing-only guid | **Superseded** |
| C′ — Fleet credential + `%SerialNumber%` on status | **Superseded for v1 status** by C″ |
| C″ — Serial + active binding + UEM ICCID (read-only) | **Accepted** (this amend) |
| Serial with **no** DeviceBinding gate | **Rejected** |
| Serial authorizing mutations | **Rejected** |

### Migration / compatibility

- PR18 `device_external_id + credential` remains supported through migration and as rollback.
- Existing `OrganizationFleetCredential` / fleet status auth shape (if already shipped) is **non-normative for v1** after this amend; do not require fleet keys in UEM App Config for status. Removal or deprecation of fleet status auth is a follow-up implementation PR, not this docs PR.
- Deprecate PR18 device credentials in UEM only after serial status is validated on fleet devices.

### Non-goals (this docs amend)

- Implementing API / APK / web changes in this docs PR
- Making `DeviceBinding.esim` nullable
- BlackBerry UEM write sync
- Free-form serial entry in the APK
- Pairing as v1 enrollment
- Resurrecting local ICCID without a new spike proof
- Using serial-only status for any mutating API

## Consequences

### Accepted Option C″

- Normative v1 App Config for status: only `roamkit.device_serial=%SerialNumber%`.
- Normative gate: active `DeviceBinding` for that serial; then UEM ICCID → team-Account `Esim` → read-only snapshot.
- Explicit tradeoff: enrolled serial knowledge ⇒ read status/usage; never mutations.
- Pairing out of v1. Fleet credentials out of v1 status App Config / status auth.
- PR18 remains fallback until explicitly deprecated after validation.
- Implementation continues in small PRs (docs ≠ API ≠ APK).

### Rejected paths

- Local ICCID / hybrid remain closed.
- ICCID-as-auth and guid-alone-as-auth remain forbidden.
- Serial without binding gate remains forbidden.
- Serial as mutation auth remains forbidden.

## Stop rule

Now that this ADR is **Accepted** (as amended for C″):

- Implementation **must** follow Option C″ for v1 managed-device status (or open a new ADR discussion first).
- Do **not** require `fleet_*` App Config keys for v1 status.
- Do **not** accept serial status without an active `DeviceBinding` for that serial.
- Do **not** authorize any mutation from serial (or from this status endpoint family).
- Do **not** treat UEM ICCID or UEM `guid` alone as sufficient authorization.
- Do **not** treat `iccid=null` / `sims=[]` as proof of no eSIM or as a trigger to mutate inventory.
- Do **not** implement local-ICCID status paths without a new ADR.
- Do **not** reintroduce pairing or fleet-credential status auth as required v1 without a new ADR discussion.
- Do **not** assume `query=serialNumber` on this tenant without re-validation.

Changing this Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
