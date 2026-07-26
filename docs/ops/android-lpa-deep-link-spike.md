# Android LPA deep link spike

Evidence pack for **PR B1**: can a GSMA `LPA:1$SM-DP+$ActivationCode` URI open
the system eSIM installer from a mobile browser?

Related code: `roamkit-web` `buildLpaUri`, `launchInstallAction`,
`NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` (default **off**).

## Security

Do **not** paste Activation Codes, SM-DP+ hosts, or full LPA URIs into this doc,
tickets, telemetry, or screenshots that leave the device.

Record only: device, browser, OS, scheme (`lpa` / `intent`), pass/fail, notes.

## DoD matrix

| Device | Browser | OS / build | Scheme | Result (pass / fail / acceptable) | Notes |
|--------|---------|------------|--------|-----------------------------------|-------|
| Samsung One UI 6 | Chrome | | `lpa` | | |
| Samsung One UI 6 | Samsung Internet | | `lpa` | | |
| Samsung One UI 7 | Chrome | | `lpa` | | |
| Samsung One UI 7 | Samsung Internet | | `lpa` | | |
| Pixel Android 15 | Chrome | | `lpa` | | |
| Xiaomi HyperOS | Chrome | | `lpa` | | |

Optional follow-up rows if `lpa` fails on Chrome:

| Device | Browser | Scheme | Result | Notes |
|--------|---------|--------|--------|-------|
| … | Chrome | `intent` | | Only if `lpa` blocked |

### Pass criteria

- **Samsung rows:** opens system eSIM / add-profile installer.
- **Pixel / Xiaomi:** **acceptable** if installer opens **or** safe no-op (browser stays on RoamKit without broken state).

### Success / failure heuristics (for product UX)

- **Success:** page becomes hidden / app backgrounds within ~2–3s; user can return.
- **Failure:** browser stays on RoamKit, unknown protocol, or no visibility change in ~2–3s.

## How to probe (flag on)

1. Build web with `NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK=1`.
2. Open setup for an owned eSIM on the device browser.
3. Pick manufacturer → **Install eSIM** (when registry allows `deep-link`).
4. Fill matrix row; never commit secrets.

## Decision Log

```text
Spike Result

Decision:
□ Ship B2
□ Extend spike
□ Cancel feature

Reason:


Enable production flag only if (when shipping B2 / later prod):
- Samsung pass rate >95% on matrix sample
- No critical browser bugs
- Fallback (guide + QR) confirmed
```

Fill this section after matrix runs. B1 is incomplete without a Decision.

## Kill switch

`NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` — leave unset/`0` in staging and production
until Decision = Ship B2 **and** enable criteria above are met. Disable immediately
if One UI / browser regresses.
