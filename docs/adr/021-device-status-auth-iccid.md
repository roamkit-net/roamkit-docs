# ADR 021: Device status auth (ICCID lookup vs credentials)

| Field | Value |
|-------|-------|
| Status | Proposed |
| Date | 2026-08 |
| Deciders | Solo operator (design lock before any ICCID / fleet-credential / UEM-read status implementation) |
| Relates to | [ADR 020](./020-organization-team-accounts.md) |

## Context

[ADR 020](./020-organization-team-accounts.md) reserved a device / UEM extension surface:

- `DeviceBinding` attaches to Account-owned eSIMs via a RoamKit-issued `device_external_id`
- A managed device profile carries that id; the client calls a status API
- No BlackBerry UEM admin sync in v1 of ADR 020

That shape is already shipped (PR18): public `POST /api/v1/device/status/` authenticates with `device_external_id` + opaque per-binding credential (hash at rest; plaintext only at issue/rotate). `device_external_id` is a lookup key only — never sufficient authorization.

PR18 is **confirmed end-to-end** on a real BlackBerry-managed Pixel 6a (UEM managed config → APK → status API → active `DeviceBinding` → usage snapshot).

Operators want simpler enrollment than per-device managed credentials. Early candidates assumed the APK could discover the active SIM **ICCID locally**. That assumption was tested and **failed** on the validated setup (see Spike result below). A later read-only UEM identity spike locked the preferred **device mapping key** (UEM `device.guid`) but did **not** clear Accept of option C (lifecycle + dual-SIM rules remain open).

Hard rules that remain unchanged:

1. **ICCID must never become a credential.** No unauthenticated `GET /device/status/{iccid}` (or body with ICCID alone).
2. **Auth and lookup stay separate.** Credential / device identity = auth; ICCID (when used) = lookup only.
3. **`Esim.account` remains the ownership boundary.**
4. **No API / APK / web implementation** of a new ICCID-based status path until this ADR is Accepted under an updated option.

## Spike results

### Local ICCID (negative proof)

Validated on BlackBerry UEM Cloud tenant device (Pixel 6a, Android 16, work profile), `roamkit-device` ICCID spike (`develop`, ADR 021 proof screen):

| Source | Result |
|--------|--------|
| BlackBerry UEM device report / REST | ICCID **visible** (e.g. `8900424101001825931`) |
| APK local read (active/default data subscription via `SubscriptionInfo.getIccId()`) | **ICCID not readable** (fail-closed with an explicit reason) |

Therefore:

```text
BlackBerry UEM sees ICCID
APK on Pixel 6a / Android 16 cannot read ICCID locally
```

**Local-ICCID hybrid is not viable on this validated setup** and **must not be Accepted** in that form. Android documentation already flags ICCID access as restricted ([SubscriptionInfo](https://developer.android.com/reference/kotlin/android/telephony/SubscriptionInfo), [unique identifier best practices](https://developer.android.com/identity/user-data-ids)).

A separate UEM REST spike also showed non-Dynamics Android **app config create/update is not supported** via API — relevant to ops pain of per-device managed values, not to ICCID readability.

### UEM identity mapping (read-only)

Read-only OAuth spike against tenant `S31564560` (`GET /api/v1/devices`, `MDMBWS.All`):

| Finding | Detail |
|---------|--------|
| Preferred mapping key | UEM **`device.guid`** (UEM-native UUID) |
| Sample uniqueness | `guid` present and unique on **100/100** listed devices |
| Consistency | Same `guid` on device list entry and nested `userDevice.device.guid` |
| ICCID as device id | **Rejected** — tenant had a **duplicate ICCID on two devices**; ICCID is SIM lookup only |
| Dual-SIM | Top-level `iccid` is always one of `sims[].iccid`, but **not always `sims[0]`** |
| REST detail by guid | On this tenant, `GET /devices/{guid}` returned **404** |

Validated read path for option C (until BlackBerry documents otherwise on this tenant):

```text
GET /api/v1/devices
      ↓
match device.guid == DeviceBinding.uem_device_guid
      ↓
resolve authoritative SIM / ICCID from that record
```

**Not proven yet:** whether `device.guid` survives wipe / re-enroll / policy refresh. Short dual REST reads showed stable IDs; lifecycle churn was not exercised.

Secondary identifiers (`udid`, `serialNumber`, `imei`, `userDevice.guid` / `activeSyncId`) were observed but are **not** the preferred RoamKit ↔ UEM map key.

## Decision (Proposed)

### Current contract (unchanged until Accept)

**Option A — PR18 per-device credential** remains the shipped, supported, and fallback device status contract:

```text
device_external_id  = lookup
binding credential  = auth
Esim.account        = ownership boundary
DeviceBinding       = association + audit + revoke
```

### Invariants (all options)

```text
ICCID            = lookup key only (never auth) when used
credential /
  device identity = auth
Esim.account     = ownership / authorization boundary (team Account)
DeviceBinding    = inventory/device association + audit + revoke boundary
                   (not torn down by this ADR)
UEM device.guid  = preferred external map to UEM device (option C)
                   — not an auth secret by itself
```

### Options compared

| Dimension | A: PR18 (current) | B: Pure fleet + local ICCID | Local hybrid (former preferred) | C: UEM-sourced ICCID (preferred Proposed direction) |
|-----------|-------------------|-----------------------------|----------------------------------|-----------------------------------------------------|
| Lookup | `device_external_id` | local ICCID | local ICCID | **ICCID from UEM REST** after `device.guid` match |
| Auth | binding credential | fleet secret alone | fleet secret | authenticated device identity (exact shape TBD) |
| Extra gate | — | none | active DeviceBinding for Esim | team Account + `DeviceBinding.uem_device_guid` |
| Android local ICCID | no | required | required | **not required** |
| UEM server read | no | no | no | **yes (read-only)** |
| UEM write / app-config automation | no | no | no | **no** (out of scope here) |
| Blast radius | one binding | all team ICCIDs | bound eSIMs only | scoped by auth + Account + binding/UEM map |
| Status on validated Pixel 6a | **works** | blocked (no local ICCID) | **not viable** | candidate (not implemented) |

**Pure B is not a recommended Accept target.**  
**Local hybrid is not an Accept target** after negative proof.  
**Preferred Proposed direction is C.** Mapping key is locked below; Accept of C is **not** cleared yet.

### Preferred mapping key (locked while Proposed)

```text
Preferred mapping key: UEM device.guid
Stored on RoamKit side (name indicative): DeviceBinding.uem_device_guid
```

Rationale from the identity spike:

- UEM-native
- Present on every sampled device
- Unique in the 100-device sample
- Consistent between device list and `userDevice.device`

`udid` remains a possible secondary / correlation field only — not the preferred stored map key.

ICCID must **not** be used as the `DeviceBinding ↔ UEM device` identity (duplicate ICCID observed across two devices; SIM swaps would rebind the wrong physical device).

Field name `uem_device_guid` is indicative for design discussion; schema is **not** introduced until Accept + an implementation PR.

### Option C — UEM-sourced ICCID (preferred Proposed direction)

```text
APK credential / device identity
      ↓
RoamKit authenticates DeviceBinding
      ↓
DeviceBinding.uem_device_guid
      ↓
GET /api/v1/devices   (validated path; see REST constraint)
      ↓
match device.guid
      ↓
resolve authoritative SIM / ICCID   (rule TBD — Accept gate)
      ↓
Esim found by ICCID AND Esim.account == Organization.team Account?
      ↓
active DeviceBinding allow / revoke gate
      ↓
return status
```

```text
UEM device.guid        = DeviceBinding ↔ UEM device map
UEM ICCID              = lookup only
credential / device id = auth
Esim.account           = ownership
DeviceBinding          = association + audit + revoke
```

Even a valid credential must not authorize arbitrary ICCIDs on the team Account without the agreed allow/revoke boundary.

### REST constraint (validated tenant)

On tenant `S31564560` (`p07003.cp1.uem.blackberry.com`):

> `GET /devices/{guid}` is **not** available (**404**). The validated read path is **list devices + match by `guid`** (or follow `userDevice` link and read nested `device`).

Implementation of option C must not assume a working detail-by-guid endpoint unless a later spike proves otherwise on the target environment.

### Dual-SIM / multi-eSIM

For **local** ICCID options (B / local hybrid): active/default data subscription only; fail closed if ambiguous. Those options are not Accept targets after negative proof.

For **option C** (still open — Accept gate):

- Do **not** assume top-level `iccid == sims[0].iccid` (tenant counterexample: dual-SIM device where top-level matched the **second** SIM)
- Top-level `iccid`, when present, was always **one of** the `sims[].iccid` values in the sample
- Some devices had empty `sims` / missing top-level `iccid` → **fail closed**
- Authoritative selection rule (e.g. prefer top-level when present and in `sims[]`, else explicit policy) must be written before Accept of C

### No Subscription ID fallback

Android Subscription ID is **not** a global RoamKit identity and **must not** map to `Esim`.

### Fleet / device credential lifecycle (if C or any fleet-scoped auth is Accepted)

- Hash or encrypt at rest per chosen secret model
- Plaintext only at issue / rotate
- Track `issued_at`, `rotated_at`, optionally `revoked_at`
- Rotation: instant cutover **or** short overlap (exactly one at Accept)
- Audit events for issue / rotate / revoke
- Never returned from a normal GET

### Migration / compatibility

- PR18 `device_external_id + credential` remains current and fallback through any migration
- Rollback must not require re-enrolling every device
- Deprecate PR18 device credentials only after a chosen replacement is validated on fleet devices

### Accept prerequisites

**Must not Accept B or local hybrid** unless a future spike overturns the negative proof on real BlackBerry-managed devices (currently blocked).

**Must not Accept C** until all of the following are confirmed:

1. **`device.guid` lifecycle** through wipe / re-enroll / policy refresh is proven, **or** behaviour when GUID changes is precisely defined (remap / re-bind / fail closed)
2. **Dual-SIM / missing-ICCID** authoritative ICCID selection + fail-closed rules for UEM-reported SIMs are specified
3. Server-side UEM read of ICCID via the validated path (`GET /devices` + match `guid`) remains viable for the mapped device
4. Auth shape for the APK call is specified (what credential / identity the APK sends — `uem_device_guid` is **not** sufficient alone)
5. Team Account ownership check on resolved `Esim`
6. Allow/revoke boundary (active `DeviceBinding`) is specified
7. PR18 migration / rollback path remains defined

Accepting **A only** (keep PR18 long-term) requires no UEM ICCID work.

### Non-goals

- Implementing option C (or any ICCID status API) in this docs step
- Schema for `DeviceBinding.uem_device_guid` before Accept
- Deleting or replacing `DeviceBinding`
- BlackBerry UEM **write** sync / automated non-Dynamics app-config write
- Web enrollment UI (`/me/orgs`) as part of this ADR Accept

## Consequences

### While Proposed

- PR18 remains the only supported device status auth path
- No new ICCID-based or UEM-lookup status implementation in `roamkit-api`, `roamkit-device`, or `roamkit-web`
- Mapping key preference is locked to UEM `device.guid`; option C remains **not** Accept-ready
- Local ICCID spike UI may remain as negative-proof tooling; it does not unlock Accept of local hybrid

### If Accepted as A

- No public device status contract change
- Operator UX for binding issue/rotate + manual UEM paste remains the product path

### If Accepted as C

- New server-side flow: authenticated `DeviceBinding` → UEM read by `uem_device_guid` → authoritative ICCID → `Esim` on team Account → status
- Requires cleared lifecycle + dual-SIM Accept gates and APK auth shape
- PR18 retained through migration

### If someone proposes resurrecting local hybrid

- Requires new positive ICCID proof on real BlackBerry-managed devices and an explicit ADR update — the Pixel 6a / Android 16 negative proof stands until then

## Stop rule

Until this ADR is **Accepted**:

- do not implement ICCID auth paths (local or UEM-sourced)
- do not add org/fleet device credentials for status API
- do not add `uem_device_guid` schema or status contract changes for option C
- do not change the PR18 public contract in a breaking way for this redesign
- do not silently treat UEM ICCID (or UEM `guid` alone) as sufficient authorization

Changing an Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
