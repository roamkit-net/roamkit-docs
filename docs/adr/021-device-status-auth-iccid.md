# ADR 021: Device status auth (ICCID lookup vs credentials)

| Field | Value |
|-------|-------|
| Status | Proposed |
| Date | 2026-08 |
| Deciders | Solo operator (design lock before any ICCID / fleet-credential implementation) |
| Relates to | [ADR 020](./020-organization-team-accounts.md) |

## Context

[ADR 020](./020-organization-team-accounts.md) reserved a device / UEM extension surface:

- `DeviceBinding` attaches to Account-owned eSIMs via a RoamKit-issued `device_external_id`
- A managed device profile carries that id; the client calls a status API
- No BlackBerry UEM admin sync in v1 of ADR 020

That shape is already shipped (PR18): public `POST /api/v1/device/status/` authenticates with `device_external_id` + opaque per-binding credential (hash at rest; plaintext only at issue/rotate). `device_external_id` is a lookup key only — never sufficient authorization.

Operators want a simpler UEM configuration: one shared fleet secret in managed config, while the APK discovers which eSIM is on the device by reading the **active data SIM ICCID** locally. That would avoid per-device managed values.

Two constraints make this a design decision, not a casual API tweak:

1. **ICCID must never become a credential.** An unauthenticated `GET /device/status/{iccid}` (or body with ICCID alone) would leak fleet eSIM status to anyone who learns an ICCID.
2. **Local ICCID read is not proven on our stack.** On modern Android, `SubscriptionInfo.getIccId()` is restricted (privileged access, carrier privileges, or limited device-/profile-owner scenarios with `READ_PHONE_STATE`; profile-owner access is deprecated). See [SubscriptionInfo](https://developer.android.com/reference/kotlin/android/telephony/SubscriptionInfo) and [unique identifier best practices](https://developer.android.com/identity/user-data-ids). A BlackBerry UEM spike also showed REST cannot create/update non-Dynamics Android app configs — which favors a single shared managed secret operationally, but does not prove ICCID readability on managed devices.

This ADR compares auth options and locks guardrails. **No API, APK, or web implementation of an ICCID auth path may ship until this ADR is Accepted** (and, for B/hybrid, until Accept prerequisites below are met).

## Decision (Proposed)

### Current contract (unchanged until Accept)

**Option A — PR18 per-device credential** remains the shipped and supported device status contract:

```text
device_external_id  = lookup
binding credential  = auth
Esim.account        = ownership boundary
DeviceBinding       = association + audit + revoke
```

### Invariants (all options)

```text
ICCID            = lookup key only (never auth)
credential       = auth
Esim.account     = ownership / authorization boundary (team Account)
DeviceBinding    = inventory/device association + audit + revoke boundary
                   (not torn down by this ADR)
```

### Options compared

| Dimension | A: Per-device credential (PR18 / current) | B: Pure fleet credential + ICCID | Hybrid (preferred Accept candidate) |
|-----------|------------------------------------------|----------------------------------|-------------------------------------|
| Lookup | `device_external_id` | ICCID within team Account | ICCID within team Account |
| Auth | binding-scoped secret | org/fleet-scoped secret alone | org/fleet-scoped secret |
| Extra gate | — | none | **active DeviceBinding for that Esim** |
| UEM config | per-device id + credential | `fleet_credential` only | `fleet_credential` only |
| Blast radius | one binding | all team Account ICCIDs | only **bound** eSIMs under that fleet secret |
| Rotation | rotate one binding | rotate fleet → all devices | rotate fleet → all devices; revoke still via unbind |
| Revocation | unbind / rotate binding | fleet rotate only | unbind DeviceBinding (and/or fleet rotate) |
| Audit | `DeviceBindingEvent` | new fleet audit required | DeviceBinding + fleet credential events |
| Android ICCID required | no | yes | yes |
| Fits UEM write limits | manual per-device (painful) | one shared managed value | one shared managed value |

**Pure B is not the recommended Accept target.** A valid fleet secret must not authorize status for arbitrary ICCIDs on the team Account without an active binding.

### Preferred Accept candidate: hybrid

```text
fleet credential
      +
ICCID (active/default data subscription)
      ↓
Organization fleet credential valid?
      ↓
Esim found by ICCID AND Esim.account == Organization.team Account?
      ↓
active DeviceBinding exists for that Esim?
      ↓
return status
```

```text
ICCID            = lookup
fleet credential = auth
Esim.account     = ownership
active DeviceBinding = allow / revoke boundary
```

Even a correct fleet secret does **not** grant status for an unbound ICCID that happens to sit on the team Account.

### Dual-SIM / multi-eSIM (normative for B/hybrid)

“Active SIM ICCID” means the ICCID of the **active/default data subscription** only:

- not the first SIM Android returns
- not the voice or SMS default unless it is also the data default
- not an arbitrary ICCID from a multi-profile list

If there is no unambiguous active/default data subscription → **fail closed** (status unavailable). No random selection.

### No Subscription ID fallback

Android may recommend Subscription ID for some local use cases because ICCID access is restricted. Subscription ID is **not** a global RoamKit identity and **must not** map to `Esim`.

- No automatic `subscriptionId → RoamKit Esim` fallback
- If ICCID is unavailable, B/hybrid is unavailable on that device
- PR18 `device_external_id + credential` remains the current / fallback contract

### Fleet credential lifecycle (minimum if hybrid or B is Accepted)

- Store hash or encrypted-at-rest per the chosen org secret model
- Plaintext only at issue / rotate
- Track `issued_at`, `rotated_at`, and optionally `revoked_at`
- Rotation rule chosen at Accept: **instant cutover** or a **short overlap window** (exactly one)
- Audit events for issue / rotate / revoke
- Credential is never returned from a normal GET

### Migration / compatibility (if hybrid or B is Accepted)

- Keep the existing PR18 `device_external_id + credential` endpoint during a migration period
- New APK may prefer the new contract, or rollout may use a feature flag
- Rollback must not require re-enrolling every device
- Deprecate PR18 device credentials only after real fleet validation

### Accept prerequisites (hard gate)

This ADR **must not** be Accepted for **B or hybrid** until all of the following are confirmed:

1. ICCID of the active/default data subscription is readable on a **real BlackBerry-managed** Android device via `roamkit-device`
2. Dual-SIM / multi-eSIM behavior is locked (active/default data only)
3. Fail-closed behavior when ICCID is unavailable
4. Fleet credential rotation + revocation lifecycle is specified (including cutover vs overlap)
5. Active `DeviceBinding` is mandatory in the hybrid model
6. Rollback / compatibility with the PR18 contract is defined

Accepting **A only** (keep PR18 as the long-term model) does **not** require the Android ICCID proof.

### Non-goals

- Implementing an ICCID status API or fleet credential schema in this ADR step
- Deleting or replacing `DeviceBinding`
- BlackBerry UEM admin sync / automated non-Dynamics app-config write
- Web enrollment UI (`/me/orgs`)
- Proving Android ICCID capability in the docs PR that lands this Proposed ADR (proof is a later device spike before Accept of B/hybrid)

## Consequences

### While Proposed

- PR18 remains the only supported device status auth path
- No ICCID-based or fleet-credential status implementation in `roamkit-api`, `roamkit-device`, or `roamkit-web`
- Org UI / enrollment work may continue against the PR18 binding model

### If Accepted as A

- No public device status contract change
- Later work focuses on operator UX for binding issue/rotate and manual UEM paste of per-device values

### If Accepted as hybrid

- New device status contract: ICCID lookup + fleet credential + team Account ownership + active DeviceBinding
- APK managed config tends toward a single `fleet_credential` (plus local ICCID discovery)
- PR18 endpoint retained through migration; deprecation only after fleet validation
- Android ICCID proof is a blocking prerequisite before Accept

### If Accepted as pure B

- Only if Accept explicitly overrides the hybrid preference (not recommended)
- Same ICCID proof and lifecycle gates as hybrid, without the DeviceBinding allow gate (larger blast radius)

## Stop rule

Until this ADR is **Accepted**:

- do not implement ICCID auth paths
- do not add org/fleet device credentials for status API
- do not change the PR18 public contract in a breaking way for this redesign

Changing an Accepted decision later requires a new ADR discussion — never a silent rewrite during implementation.
