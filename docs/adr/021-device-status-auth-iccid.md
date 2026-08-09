# ADR 021: Device status auth (ICCID lookup vs credentials)

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-09 |
| Amended | 2026-08-09 |
| Deciders | Solo operator |
| Relates to | [ADR 020](./020-organization-team-accounts.md) |

## Context

[ADR 020](./020-organization-team-accounts.md) reserved a device / UEM extension surface:

- `DeviceBinding` attaches to Account-owned eSIMs via a RoamKit-issued `device_external_id`
- A managed device profile carries that id; the client calls a status API
- No BlackBerry UEM admin sync in v1 of ADR 020

That shape shipped as **PR18**: public `POST /api/v1/device/status/` authenticates with `device_external_id` + opaque per-binding credential (hash at rest; plaintext only at issue/rotate). `device_external_id` is a lookup key only — never sufficient authorization.

PR18 is **confirmed end-to-end** on a real BlackBerry-managed Pixel 6a (UEM managed config → APK → status API → active `DeviceBinding` → usage snapshot).

Operators need **one** UEM App Configuration per organization — not a new managed-config secret per eSIM or per SIM swap. Local ICCID on the APK was tested and **failed**. Read-only UEM REST can supply ICCID after the physical device is identified.

Hard rules (unchanged):

1. **ICCID must never become a credential.** No unauthenticated `GET /device/status/{iccid}` (or body with ICCID alone).
2. **Auth and lookup stay separate.**
3. **`Esim.account` remains the ownership boundary.**
4. **UEM `device.guid` alone is never sufficient authorization.**
5. **Device serial alone is never sufficient authorization.**

## Spike results (historical + amend proof)

### Local ICCID (negative proof)

Validated on BlackBerry UEM Cloud tenant device (Pixel 6a, Android 16, work profile):

| Source | Result |
|--------|--------|
| BlackBerry UEM device report / REST | ICCID **visible** |
| APK local read (`SubscriptionInfo.getIccId()`) | **ICCID not readable** |

**Local-ICCID hybrid is rejected** and must not be reopened without new positive proof on real BlackBerry-managed devices.

Non-Dynamics Android **app config create/update is not supported** via UEM REST — operators cannot automate per-device App Configuration writes for this package.

### UEM App Configuration variable expansion (positive proof — amend)

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

**Conclusion:** BlackBerry UEM **does** perform per-device substitution into Android managed App Configuration for at least `%SerialNumber%`. Pairing is **not** required as the v1 path to get a stable physical-device identity onto the APK.

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

Validated read path (fleet / amend):

```text
GET /api/v1/devices
      ↓
match device.serialNumber == DeviceBinding.uem_serial_number
      ↓
use that record's current device.guid + authoritative ICCID
```

### GUID vs serial lifecycle (amend proof)

Same Pixel after re-activation:

| Field | Before | After |
|-------|--------|-------|
| `serialNumber` | `36281JEGR04531` | `36281JEGR04531` (stable) |
| `device.guid` | `bc473029-…` | `fb3de589-…` (changed) |
| ICCID | team eSIM ICCID | same ICCID after inventory refresh |

```text
device serial   = physical phone bootstrap identity (stable across re-enroll)
device.guid     = current UEM inventory record (may change)
ICCID           = current SIM state / lookup only
```

Empty telephony inventory means **UEM inventory unavailable/stale**, not “no eSIM”.

## Decision (Accepted — amended 2026-08-09)

**Option C′ — fleet auth + `%SerialNumber%` bootstrap + UEM-sourced ICCID** is the normative device-status target for managed fleets.

**Option A — PR18** remains the shipped contract and **required fallback** through migration. It must not be broken while fleet ships.

**Option B / local hybrid** remain **rejected**.

**Pairing-only `uem_device_guid` enrollment** (prior Accept wording) is **superseded** for v1 by serial bootstrap. Pairing is **not** a v1 fleet enrollment requirement.

### Role separation (normative)

```text
fleet_external_id     = find organization / fleet
fleet_credential      = authenticate fleet
device_serial         = identify physical phone (UEM %SerialNumber%; not a credential)
UEM device.guid       = current UEM inventory record (correlation / cache)
ICCID                 = find current eSIM (lookup only)
```

### Authorization boundary (hard security rule)

```text
valid fleet credential
AND
active DeviceBinding for that organization + device serial
AND
Esim.account == Organization.account
```

A leaked fleet secret **must not** authorize status for an arbitrary serial. There must be an active binding for that org + serial. Serial alone is never auth. GUID alone is never auth. ICCID alone is never auth.

### UEM App Configuration (ops lock)

**One** App Configuration per organization for `net.roamkit.bbuem`, identical on every phone:

| Key | Role |
|-----|------|
| `roamkit.fleet_external_id` | Fleet lookup |
| `roamkit.fleet_credential` | Fleet auth secret |
| `roamkit.device_serial` | Must be UEM variable **`%SerialNumber%`** (per-device expansion) |

No new managed-config values on eSIM swap. No per-eSIM credentials in UEM for the fleet path. No free-form / user-typed serial entry in the APK after managed config is present.

### DeviceBinding identity fields (normative)

| Field | Role |
|-------|------|
| `uem_serial_number` | **Stable** map key for the physical device (from App Config / UEM `serialNumber`) |
| `uem_device_guid` | **Current** UEM record correlation / cache; may be refreshed when UEM returns a new guid for the same serial |

Operator creates / maintains an active `DeviceBinding` keyed by organization + `uem_serial_number` (once per physical phone, not per eSIM). First implementation schema PR **must keep `DeviceBinding.esim` non-null** (PR18 / legacy). Nullable cleanup is a later PR after fleet is proven.

### Status resolution (fleet path)

```text
POST /api/v1/device/status/
  fleet_external_id + fleet_credential + device_serial
      ↓
verify fleet credential (current or grace previous)
      ↓
require active DeviceBinding(org, uem_serial_number == device_serial)
      ↓
UEM GET /devices + match serialNumber
      ↓
current device.guid (+ refresh cache on binding) + authoritative ICCID
      ↓
Esim on Organization.account
      ↓
status snapshot
```

Fleet path **ignores** `DeviceBinding.esim` for resolution of the current eSIM.

### Dual-SIM rule (normative)

- Use **top-level `iccid`** when present (on this tenant it is always one of `sims[].iccid`).
- Do **not** assume `iccid == sims[0].iccid`.
- If top-level missing and/or `sims=[]` → `uem_inventory_unavailable` (fail closed; no inventory mutation).

### GUID / serial lifecycle (normative)

- If UEM `device.guid` changes after wipe/re-enroll but **serial is unchanged**: RoamKit **re-discovers** the device via serial match and may update cached `uem_device_guid`. No pairing. No silent remap by ICCID.
- If serial cannot be matched in UEM inventory → fail closed (`binding_not_found` / inventory miss as appropriate); do not invent a device.
- ICCID changes (SIM swap) do **not** require App Configuration changes.

### Fleet credential rotation (normative)

Default is **short overlap**, not instant single-secret cutover:

```text
current credential   = valid
previous credential  = valid during grace / UEM rollout window
```

After the window, previous is revoked. Instant cutover that bricks the fleet when App Configuration has not reached all devices is rejected as the default.

### Distinct device API failure codes (normative)

Do **not** collapse these into one generic 404:

| Condition | Code |
|-----------|------|
| Bad/missing fleet auth; no active binding for org+serial | `binding_not_found` (or equivalent binding/auth failure — APK treats as config/binding problem) |
| UEM ICCID present, no team-Account `Esim` | `iccid_not_found` |
| UEM device resolved but telephony empty/stale; or serial not found in UEM list when inventory is otherwise expected | `uem_inventory_unavailable` (or binding/auth miss if binding absent — do not conflate) |

On all three families: **no** create/transfer/unbind/provider side effects.

### ICCID miss / UEM inventory stale (normative)

- ICCID present in UEM, no team-Account `Esim` → read-only miss (`iccid_not_found`)
- `iccid=null` / `sims=[]` → `uem_inventory_unavailable`, not “no eSIM”
- Must not mutate RoamKit inventory on the read path alone

### REST constraint (validated tenant)

On tenant `S31564560` (`p07003.cp1.uem.blackberry.com`):

> - `GET /devices/{guid}` is **not** available (**404**).
> - `query=serialNumber=…` is **not** supported (**400** Unknown query field).
> - Validated path: **`GET /devices` + match `serialNumber`** (and/or match cached guid when still present).
> - Do not assume detail-by-guid or serial query filters unless proven later on this tenant.

Optional later optimization: if a working App Config variable for IMEI/UDID is proven, `query=imei=` / `query=udid=` may replace full-list match — that does **not** change the App Config serial bootstrap lock.

### Audit events (normative minimum)

```text
fleet_credential_rotated
# binding create / revoke / serial remap events as implemented
```

Pairing audit events (`pairing_issued` / `pairing_completed` / `pairing_revoked`) are **not** required for v1 fleet (pairing is not the v1 enrollment path).

### Mapping keys

```text
APK / App Config bootstrap : roamkit.device_serial = %SerialNumber%
Stored stable map key      : DeviceBinding.uem_serial_number
Stored UEM correlation     : DeviceBinding.uem_device_guid (cache; refreshable)
Current SIM lookup         : UEM ICCID → Esim.iccid on Organization.account
```

ICCID must **not** be the DeviceBinding ↔ UEM device identity.

### Rejected / superseded options

| Option | Status |
|--------|--------|
| A — PR18 only as long-term exclusive model | Superseded as **target**; retained as **fallback** |
| B — Pure fleet + local ICCID | **Rejected** |
| Local hybrid (fleet + local ICCID + binding gate) | **Rejected** |
| C — Fleet + UEM ICCID + **pairing-only guid** | **Superseded** by C′ (serial bootstrap) |
| C′ — Fleet + `%SerialNumber%` + UEM ICCID | **Accepted** (this amend) |

### Migration / compatibility

- PR18 `device_external_id + credential` remains supported through migration and as rollback.
- Rollback must not require re-enrolling every device into a new package solely due to fleet cutover mistakes.
- Deprecate PR18 device credentials in UEM only after fleet + serial bootstrap is validated on fleet devices.
- Transitional UEM-guid-on-PR18-binding proof may remain until fleet endpoints replace it; it must not be confused with the Accepted fleet App Configuration model (`fleet_*` + `%SerialNumber%`).

### Non-goals (this Accept amend)

- Implementing fleet/status/APK/web changes in this docs PR
- Making `DeviceBinding.esim` nullable in the first implementation schema PR
- BlackBerry UEM **write** sync / automated non-Dynamics app-config write
- Free-form serial or GUID entry in the APK
- Authorizing status from fleet credential alone or serial alone
- Instant fleet-secret cutover with no grace window
- Requiring pairing as v1 enrollment
- Resurrecting local ICCID without a new spike proof
- Assuming `query=serialNumber` works without tenant re-proof

## Consequences

### Accepted Option C′

- Normative target: one UEM App Configuration per org with `fleet_external_id`, `fleet_credential`, and `device_serial=%SerialNumber%`; UEM list+match by serial; refreshable guid cache; UEM-sourced ICCID; distinct error codes; grace-period fleet rotation.
- Pairing is **out of v1** fleet enrollment.
- Implementation must follow this ADR in small PRs (schema ≠ status API ≠ APK).
- First schema PR keeps `DeviceBinding.esim` non-null and introduces stable `uem_serial_number` (+ guid cache field as specified).
- PR18 remains fallback until explicitly deprecated after validation.

### Rejected paths

- Local ICCID / hybrid remain closed by the Pixel negative proof.
- ICCID-as-auth, serial-alone-as-auth, and guid-alone-as-auth remain forbidden.
- Pairing-only guid enrollment is no longer the Accepted v1 path.

## Stop rule

Now that this ADR is **Accepted** (as amended):

- Implementation **must** follow Option C′ as specified here (or open a new ADR discussion first).
- Do **not** silently widen blast radius (fleet credential without active DeviceBinding).
- Do **not** treat UEM ICCID, UEM `guid`, or device serial alone as sufficient authorization.
- Do **not** treat `iccid=null` / `sims=[]` as proof of no eSIM or as a trigger to mutate inventory.
- Do **not** mix nullable-`esim` cleanup into the first fleet schema PR.
- Do **not** implement local-ICCID status paths without a new ADR.
- Do **not** reintroduce pairing as a required v1 path without a new ADR discussion.
- Do **not** assume `query=serialNumber` on this tenant without re-validation.

Changing this Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
