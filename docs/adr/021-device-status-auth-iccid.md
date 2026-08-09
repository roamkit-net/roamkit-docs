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

Operators want simpler enrollment than per-device managed credentials. Early candidates assumed the APK could discover the active SIM **ICCID locally**. That assumption was tested and **failed** on the validated setup (see Spike result below).

Hard rules that remain unchanged:

1. **ICCID must never become a credential.** No unauthenticated `GET /device/status/{iccid}` (or body with ICCID alone).
2. **Auth and lookup stay separate.** Credential / device identity = auth; ICCID (when used) = lookup only.
3. **`Esim.account` remains the ownership boundary.**
4. **No API / APK / web implementation** of a new ICCID-based status path until this ADR is Accepted under an updated option.

## Spike result (negative proof)

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
```

### Options compared

| Dimension | A: PR18 (current) | B: Pure fleet + local ICCID | Local hybrid (former preferred) | C: UEM-sourced ICCID (new preferred direction) |
|-----------|-------------------|-----------------------------|----------------------------------|------------------------------------------------|
| Lookup | `device_external_id` | local ICCID | local ICCID | **ICCID from UEM REST** (server-side) |
| Auth | binding credential | fleet secret alone | fleet secret | authenticated device identity (exact shape TBD) |
| Extra gate | — | none | active DeviceBinding for Esim | team Account + mapping `DeviceBinding ↔ UEM device` (identifier TBD) |
| Android local ICCID | no | required | required | **not required** |
| UEM server read | no | no | no | **yes (read-only)** |
| UEM write / app-config automation | no | no | no | **no** (out of scope here) |
| Blast radius | one binding | all team ICCIDs | bound eSIMs only | scoped by auth + Account + binding/UEM map |
| Status on validated Pixel 6a | **works** | blocked (no local ICCID) | **not viable** | candidate (not implemented) |

**Pure B is not a recommended Accept target.**  
**Local hybrid is not an Accept target** after negative proof.  
**Preferred Proposed direction is C**, pending mapping/auth decisions below.

### Option C — UEM-sourced ICCID (preferred Proposed direction)

```text
APK credential / device identity
      ↓
RoamKit authenticates device
      ↓
RoamKit ↔ UEM server-side device lookup (read-only)
      ↓
UEM returns ICCID
      ↓
Esim found by ICCID AND Esim.account == Organization.team Account?
      ↓
active DeviceBinding / allow gate (exact shape TBD with mapping)
      ↓
return status
```

```text
UEM ICCID              = lookup
credential / device id = auth
Esim.account           = ownership
DeviceBinding          = association + audit + revoke (+ UEM device map TBD)
```

Even a valid credential must not authorize arbitrary ICCIDs on the team Account without the agreed allow/revoke boundary.

### Open decision (not locked yet)

**How RoamKit identifies the same physical device inside UEM** is intentionally **undecided** in this revision. Candidates to evaluate in a follow-up spike (read-only):

- UEM device GUID
- UDID / hardware identifiers exposed by UEM REST
- other immutable identifiers available on managed devices

Do **not** invent a mapping or implement status-by-UEM-ICCID until that identifier is chosen and recorded in an Accept (or a further Proposed update).

ICCID from UEM answers “which SIM does UEM see on this device?” — it does **not** by itself answer “which RoamKit `DeviceBinding` / credential is calling?”

### Dual-SIM / multi-eSIM

For **local** ICCID options (B / local hybrid): active/default data subscription only; fail closed if ambiguous. Those options are not Accept targets after negative proof.

For **option C**: which UEM SIM field is authoritative when UEM reports multiple SIMs must be defined before Accept of C (e.g. primary / data / single reported ICCID). Fail closed if UEM returns no usable ICCID for the mapped device.

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

1. Stable, documented mapping key: `DeviceBinding` (or equivalent RoamKit device identity) ↔ UEM device
2. Server-side UEM read of ICCID works for that mapped device (OAuth client already proven for other reads)
3. Auth shape for the APK call is specified (what credential / identity the APK sends)
4. Multi-SIM / missing-ICCID fail-closed rules for UEM-reported SIMs
5. Team Account ownership check on resolved `Esim`
6. Allow/revoke boundary (active `DeviceBinding` or explicit successor) is specified
7. PR18 migration / rollback path remains defined

Accepting **A only** (keep PR18 long-term) requires no UEM ICCID work.

### Non-goals

- Implementing option C (or any ICCID status API) in this docs step
- Choosing the final UEM device identifier in this docs step without a mapping spike
- Deleting or replacing `DeviceBinding`
- BlackBerry UEM **write** sync / automated non-Dynamics app-config write
- Web enrollment UI (`/me/orgs`) as part of this ADR Accept

## Consequences

### While Proposed

- PR18 remains the only supported device status auth path
- No new ICCID-based or UEM-lookup status implementation in `roamkit-api`, `roamkit-device`, or `roamkit-web`
- Local ICCID spike UI may remain as negative-proof tooling; it does not unlock Accept of local hybrid

### If Accepted as A

- No public device status contract change
- Operator UX for binding issue/rotate + manual UEM paste remains the product path

### If Accepted as C

- New server-side flow: authenticated device → UEM ICCID read → `Esim` lookup on team Account → status
- Requires prior lock of UEM device mapping identifier and APK auth shape
- PR18 retained through migration

### If someone proposes resurrecting local hybrid

- Requires new positive ICCID proof on real BlackBerry-managed devices and an explicit ADR update — the Pixel 6a / Android 16 negative proof stands until then

## Stop rule

Until this ADR is **Accepted**:

- do not implement ICCID auth paths (local or UEM-sourced)
- do not add org/fleet device credentials for status API
- do not change the PR18 public contract in a breaking way for this redesign
- do not silently treat UEM ICCID as sufficient authorization

Changing an Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
