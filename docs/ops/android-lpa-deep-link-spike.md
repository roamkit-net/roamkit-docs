# Android LPA deep link spike

Evidence pack for **PR B1**: can a GSMA `LPA:1$SM-DP+$ActivationCode` URI open
the system eSIM installer from a mobile browser?

Related code: `roamkit-web` `buildLpaUri`, `launchInstallAction`,
`buildAndroidInstallProbes`, `NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` (default **off**).

## Security

Do **not** paste Activation Codes, SM-DP+ hosts, or full LPA URIs into this doc,
tickets, telemetry, or screenshots that leave the device.

Record only: device, browser, OS, scheme (`lpa` / `intent` / probe id), pass/fail, notes.

## DoD matrix

| Device | Browser | OS / build | Scheme / probe | Result (pass / fail / acceptable) | Notes |
|--------|---------|------------|----------------|-----------------------------------|-------|
| Samsung Z Fold 6 | Chrome | staging 2026-07-27 | `lpa` | **fail** | No visibility change; fallback OK |
| Samsung Z Fold 6 | Samsung Internet | staging 2026-07-27 | `lpa` | **fail** | Same as Chrome |
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `intent`, `intent:`, `intent %24`, `lpa:` | **fail** | Couldn't open |
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `intent-phone` (`com.android.phone`) | **fail** | Leaves browser → Play Store “Item not found” (false positive for visibility heuristic) |
| Samsung Z Fold 6 | … | | `intent-samsung` / `intent-euicc` / activate / manage-sims | | round 2 probes |
| Pixel Android 15 | Chrome | | `lpa` | | |
| Xiaomi HyperOS | Chrome | | `lpa` | | |

### Pass criteria

- **Samsung rows:** opens system eSIM / add-profile installer.
- **Not pass:** Play Store, “item not found”, chooser with no eSIM UI, or browser no-op.
- **Pixel / Xiaomi:** **acceptable** if installer opens **or** safe no-op (browser stays on RoamKit without broken state).

### Success / failure heuristics (for product UX)

- **Heuristic only:** page becomes hidden within ~2–3s ≠ product pass.
- **Product pass:** user reaches eSIM download / Add eSIM UI with LPA applied (or clear path to enter it).
- **Failure:** browser stays on RoamKit, unknown protocol, Play Store miss, or no eSIM UI.

## How to probe (flag on)

1. Staging with `NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK=1`.
2. Setup → manufacturer → tap each spike probe button.
3. Record probe id + pass/fail; never commit secrets.

## Decision Log

```text
Spike Result

Decision:
□ Ship B2
☑ Extend spike
□ Cancel feature

Reason:
2026-07-27 — Z Fold 6: LPA: and bare intent:// fail on Chrome + Samsung Internet.
intent+phone backgrounds browser but opens Play Store “item not found” — not an
eSIM installer (fail). Round 2: OEM packages (Samsung telephonyui, Google eUICC)
and SIM settings actions. Production flag stays off.

Enable production flag only if (when shipping B2 / later prod):
- Samsung pass rate >95% on matrix sample (real eSIM UI, not Play Store)
- No critical browser bugs
- Fallback (guide + QR) confirmed
```

## Kill switch

`NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` — leave unset/`0` in production; staging may
stay `1` while Extending spike. Disable immediately if One UI / browser regresses.
