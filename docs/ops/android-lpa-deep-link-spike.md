# Android LPA deep link spike

Evidence pack for **PR B1 → B2**: Android eSIM install from browser.

Related code: `roamkit-web` `buildAndroidUniversalLink`, `buildAndroidInstallActions`,
`launchInstallAction`, `NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` (default **off** in
production bake; staging may be `1`).

## Security

Do **not** paste Activation Codes, SM-DP+ hosts, or full LPA URIs into this doc,
tickets, telemetry, or screenshots that leave the device.

Record only: device, browser, OS, scheme / action id, pass/fail, notes.

## DoD matrix

| Device | Browser | OS / build | Scheme / probe | Result | Notes |
|--------|---------|------------|----------------|--------|-------|
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `lpa` / bare `intent://` | **fail** | No installer |
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `intent-phone` / Google eUICC packages | **fail** | Play Store miss / listing |
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `android-universal` (encoded) | **pass** | System “Postavite eSIM” → profile check |
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `android-universal-raw` | **pass** | Same as encoded; dropped as duplicate |
| Samsung Z Fold 6 | Chrome / SI | staging 2026-07-27 | `settings-network-dashboard` | **acceptable** | Opens Connections (Veze); secondary CTA |
| Samsung Z Fold 6 | … | | other Settings / Manage SIMs | **fail** | Dropped |
| Pixel Android 15 | Chrome | | `android-universal` | | follow-up |
| Xiaomi HyperOS | Chrome | | `android-universal` | | follow-up |

### Pass criteria

- **Samsung:** opens system eSIM / add-profile installer with LPA applied.
- **Not pass:** Play Store, “item not found”, or browser no-op.
- **Settings bridge:** acceptable as secondary path only (user still uses QR/manual).

### Success / failure heuristics (product UX)

- Universal HTTPS often shows a **system dialog over the page** without
  `visibilityState === hidden` — do **not** treat missing visibility as failure
  for `https` actions.
- Product pass = user sees Set up eSIM / profile download UI.

## Decision Log

```text
Spike Result

Decision:
☑ Ship B2
□ Extend spike
□ Cancel feature

Reason:
2026-07-27 — Z Fold 6 confirmed: esimsetup.android.com universal link opens
native eSIM setup (encoded carddata). Network dashboard is secondary
Connections bridge. Drop LPA:/intent package probes and universal-raw duplicate.

Product CTAs (flagged):
1. Install eSIM → Android universal HTTPS
2. Open Connections settings → NetworkDashboardActivity

Enable production flag only if:
- Samsung pass rate >95% on broader matrix (Pixel/Xiaomi follow-up OK as later)
- No critical browser bugs
- Fallback (guide + QR) confirmed — already on setup page
```

## Kill switch

`NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` — production bake stays empty until explicit
enable. Staging may keep `1`. Disable immediately if One UI / GMS regresses.
