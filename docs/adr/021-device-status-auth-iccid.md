# ADR 021: Device status auth (ICCID lookup vs credentials)

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-09 |
| Deciders | Solo operator |
| Relates to | [ADR 020](./020-organization-team-accounts.md) |

## Context

[ADR 020](./020-organization-team-accounts.md) reserved a device / UEM extension surface:

- `DeviceBinding` attaches to Account-owned eSIMs via a RoamKit-issued `device_external_id`
- A managed device profile carries that id; the client calls a status API
- No BlackBerry UEM admin sync in v1 of ADR 020

That shape shipped as **PR18**: public `POST /api/v1/device/status/` authenticates with `device_external_id` + opaque per-binding credential (hash at rest; plaintext only at issue/rotate). `device_external_id` is a lookup key only — never sufficient authorization.

PR18 is **confirmed end-to-end** on a real BlackBerry-managed Pixel 6a (UEM managed config → APK → status API → active `DeviceBinding` → usage snapshot).

Operators need **one** UEM App Configuration per organization — not a new managed-config secret per eSIM or per SIM swap. Local ICCID on the APK was tested and **failed**. Read-only UEM REST can supply ICCID after mapping by UEM `device.guid`.

A staging/production **UEM read proof** (optional `DeviceBinding.uem_device_guid` + list/match devices + ICCID → team-Account `Esim`) already exists behind PR18 auth. That proof is **not** the Accepted fleet enrollment model; it remains a transitional capability until fleet auth + pairing ship.

Hard rules (unchanged):

1. **ICCID must never become a credential.** No unauthenticated `GET /device/status/{iccid}` (or body with ICCID alone).
2. **Auth and lookup stay separate.**
3. **`Esim.account` remains the ownership boundary.**
4. **UEM `device.guid` alone is never sufficient authorization.**

## Spike results (historical)

### Local ICCID (negative proof)

Validated on BlackBerry UEM Cloud tenant device (Pixel 6a, Android 16, work profile):

| Source | Result |
|--------|--------|
| BlackBerry UEM device report / REST | ICCID **visible** |
| APK local read (`SubscriptionInfo.getIccId()`) | **ICCID not readable** |

**Local-ICCID hybrid is rejected** and must not be reopened without new positive proof on real BlackBerry-managed devices.

Non-Dynamics Android **app config create/update is not supported** via UEM REST — relevant to why pairing (not automated per-device app-config write) is the enrollment path.

### UEM identity mapping (read-only)

Tenant `S31564560` (`GET /api/v1/devices`, `MDMBWS.All`):

| Finding | Detail |
|---------|--------|
| Preferred mapping key | UEM **`device.guid`** |
| Sample uniqueness | Present and unique on **100/100** listed devices |
| ICCID as device id | **Rejected** (duplicate ICCID across two devices observed) |
| Dual-SIM | Top-level `iccid` ∈ `sims[]`, but **not always** `sims[0]` |
| REST detail by guid | `GET /devices/{guid}` returned **404** on this tenant |

Validated read path:

```text
GET /api/v1/devices
      ↓
match device.guid == DeviceBinding.uem_device_guid
      ↓
resolve authoritative SIM / ICCID from that record
```

### UEM eSIM swap observation (read-only)

Same Pixel (`device.guid` unchanged) after eSIM replace: guid stable; ICCID changed after UEM inventory refreshed; intermediate `iccid=null` / `sims=[]` observed.

```text
device.guid = device identity
ICCID       = current SIM state / lookup only
```

Empty telephony inventory means **UEM inventory unavailable/stale**, not “no eSIM”.

## Decision (Accepted)

**Option C — fleet auth + UEM-sourced ICCID** is the normative device-status target for managed fleets.

**Option A — PR18** remains the shipped contract and **required fallback** through migration. It must not be broken while fleet ships.

**Option B / local hybrid** are **rejected**.

### Role separation (normative)

```text
fleet_external_id     = find organization / fleet
fleet_credential      = authenticate fleet
uem_device_guid       = identify physical UEM device (from pairing only)
ICCID                 = find current eSIM (lookup only)
```

### Authorization boundary (hard security rule)

```text
valid fleet credential
AND
active DeviceBinding(organization, uem_device_guid)
AND
Esim.account == Organization.account
```

A leaked fleet secret **must not** authorize status for an arbitrary GUID. There must be an active binding for that org + guid. GUID alone is never auth.

### UEM App Configuration (ops lock)

**One** App Configuration per organization for `net.roamkit.bbuem`, identical on every phone:

| Key | Role |
|-----|------|
| `roamkit.fleet_external_id` | Fleet lookup |
| `roamkit.fleet_credential` | Fleet auth secret |

No new managed-config values on eSIM swap. No per-eSIM credentials in UEM for the fleet path.

### Device enrollment + pairing (normative)

- Operator creates `DeviceBinding` with `uem_device_guid` (once per physical UEM device, not per eSIM).
- RoamKit issues a **pairing code** bound to that exact binding.
- APK completes pairing once; server returns that binding’s `uem_device_guid`.
- APK persists the server-issued guid locally and sends it on status calls.
- APK **must not** accept a free-form / user-typed GUID after pairing (blocks GUID spray with a stolen fleet secret).

Pairing code rules:

- Single-use — successful complete invalidates immediately
- Short-lived expiry
- Bound 1:1 to one `DeviceBinding` (cannot change org or `uem_device_guid`)
- No reuse of an old code; re-pair requires a newly issued code
- Pairing endpoint rate-limited / brute-force protected
- Hashed at rest; plaintext only at issue

Re-pair is for app wipe / new phone / guid remap — **not** for SIM swap.

### Status resolution (fleet path)

```text
POST /api/v1/device/status/
  fleet_external_id + fleet_credential + uem_device_guid
      ↓
verify fleet credential (current or grace previous)
      ↓
require active DeviceBinding(org, uem_device_guid)
      ↓
UEM GET /devices + match guid
      ↓
authoritative ICCID
      ↓
Esim on Organization.account
      ↓
status snapshot
```

Fleet path **ignores** `DeviceBinding.esim` for resolution. First implementation schema PR **must keep `DeviceBinding.esim` non-null** (PR18 / legacy). Nullable cleanup is a later PR after fleet is proven.

### Dual-SIM rule (normative)

- Use **top-level `iccid`** when present (on this tenant it is always one of `sims[].iccid`).
- Do **not** assume `iccid == sims[0].iccid`.
- If top-level missing and/or `sims=[]` → `uem_inventory_unavailable` (fail closed; no inventory mutation).

### GUID lifecycle (normative)

If UEM `device.guid` changes after wipe/re-enroll: **fail closed** until operator remaps binding and issues a **new** pairing code. No silent remap by ICCID.

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
| Bad/missing fleet auth; no active binding for org+guid; unpaired / invalid pair state | `binding_not_found` (or equivalent binding/auth failure — APK treats as config/binding problem) |
| UEM ICCID present, no team-Account `Esim` | `iccid_not_found` |
| UEM device resolved but telephony empty/stale | `uem_inventory_unavailable` |

On all three: **no** create/transfer/unbind/provider side effects.

### ICCID miss / UEM inventory stale (normative)

Unchanged from prior locks:

- ICCID present in UEM, no team-Account `Esim` → read-only miss (`iccid_not_found`)
- `iccid=null` / `sims=[]` → `uem_inventory_unavailable`, not “no eSIM”
- Must not mutate RoamKit inventory on the read path alone

### REST constraint (validated tenant)

On tenant `S31564560` (`p07003.cp1.uem.blackberry.com`):

> `GET /devices/{guid}` is **not** available (**404**). Implementation must use **list devices + match by `guid`** (or equivalent validated path). Do not assume detail-by-guid unless proven later.

### Audit events (normative minimum)

```text
pairing_issued
pairing_completed
pairing_revoked          # explicit revoke and/or expired (reason distinguished)
fleet_credential_rotated
```

Full ops UI is not required for Accept; events are.

### Mapping key

```text
Preferred mapping key: UEM device.guid
Stored on RoamKit: DeviceBinding.uem_device_guid
```

ICCID must **not** be the DeviceBinding ↔ UEM device identity.

### Rejected options

| Option | Status |
|--------|--------|
| A — PR18 only as long-term exclusive model | Superseded as **target**; retained as **fallback** |
| B — Pure fleet + local ICCID | **Rejected** |
| Local hybrid (fleet + local ICCID + binding gate) | **Rejected** |
| C — Fleet + UEM ICCID + pairing | **Accepted** |

### Migration / compatibility

- PR18 `device_external_id + credential` remains supported through migration and as rollback.
- Rollback must not require re-enrolling every device into a new package solely due to fleet cutover mistakes.
- Deprecate PR18 device credentials in UEM only after fleet + pairing is validated on fleet devices.
- Transitional UEM-guid-on-PR18-binding proof may remain until fleet endpoints replace it; it must not be confused with the Accepted fleet App Configuration model.

### Non-goals (this Accept)

- Implementing fleet/pairing/status changes in this docs PR
- Making `DeviceBinding.esim` nullable in the first implementation schema PR
- BlackBerry UEM **write** sync / automated non-Dynamics app-config write
- Free-form GUID entry in the APK
- Authorizing status from fleet credential alone
- Instant fleet-secret cutover with no grace window
- Web full enrollment UI as part of Accept (admin/API pairing issue is enough for v1)
- Resurrecting local ICCID without new spike proof

## Consequences

### Accepted Option C

- Normative target: one UEM App Configuration per org; pairing once per phone; UEM-sourced ICCID; distinct error codes; grace-period fleet rotation; audit events above.
- Implementation must follow this ADR in small PRs (schema ≠ status API ≠ APK).
- First schema PR keeps `DeviceBinding.esim` non-null.
- PR18 remains fallback until explicitly deprecated after validation.

### Rejected paths

- Local ICCID / hybrid remain closed by the Pixel negative proof.
- ICCID-as-auth and guid-alone-as-auth remain forbidden.

## Stop rule

Now that this ADR is **Accepted**:

- Implementation **must** follow Option C as specified here (or open a new ADR discussion first).
- Do **not** silently widen blast radius (fleet credential without active DeviceBinding).
- Do **not** treat UEM ICCID or UEM `guid` alone as sufficient authorization.
- Do **not** treat `iccid=null` / `sims=[]` as proof of no eSIM or as a trigger to mutate inventory.
- Do **not** mix nullable-`esim` cleanup into the first fleet schema PR.
- Do **not** implement local-ICCID status paths without a new ADR.

Changing this Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
