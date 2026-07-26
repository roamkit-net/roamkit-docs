# RFC 002: Post-purchase eSIM onboarding

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Authors | Product / Engineering |

## Summary

Guided post-purchase setup so users install, enable, and confirm an eSIM without being dropped on a raw detail page. Complements [RFC 001](./001-self-service-esim-flow.md) (purchase → usage → top-up) with Wave 1 product UX locked by [ADR 014](../adr/014-esim-lifecycle-install-telemetry.md).

## Goals

- Onboarding wizard after successful purchase.
- Device-aware install UX (iPhone / Android / Desktop).
- Activation-policy warning from purchase-time snapshot on `Esim`.
- Install telemetry funnel + ownership-safe event trail.
- Lifecycle statuses with enforced transitions.

## Non-goals (Wave 1)

- Usage auto-refresh / Celery usage poller / daily usage estimates.
- Notification / email engine.
- Live connectivity test or “Generate support report” UI.
- Smart destination recommendations.
- Native home-screen widget (requires mobile app).
- Detecting Android manufacturer from User-Agent.
- Per-model install guides (manufacturer-level guides only).
- Persisting manufacturer choice in URL or localStorage.
- Backend-hosted Android guide content (Wave 1 uses a frontend TypeScript registry).

## User flow

Route: `/me/esims/[id]/setup`

Steps:

1. **Install eSIM** — device-aware:
   - **Desktop:** QR primary (“Open Camera on your phone”); optional **Phone instructions** opens the same Android manufacturer picker.
   - **iPhone:** Apple direct install link when available; QR / manual codes as fallback.
   - **Android:** manufacturer picker (stable slug ids: `samsung`, `pixel`, `other`, …) then registry guide steps, plus QR / LPA / SM-DP+ codes.
2. **Enable eSIM** — static checklist.
3. **Turn on Data Roaming** — static network helper (no live probe).
4. **Confirm everything works** — activation-policy banner from `esim.activation_policy`.

### Android install guides (registry)

Guide **registry** is the single source of truth for Android installation instructions. UI components render registry content and must not contain manufacturer-specific installation logic.

- Stable slug `id`s (immutable; not derived from display title).
- UI talks to a **GuideService** boundary (list / get / resolve); components do not import raw registry data.
- Wave 1 storage: TypeScript content module in the web app.
- Later (optional): move stabilized guides to an API content registry (YAML/JSON + REST), then Admin/DB only if ops need non-engineer editing.

Manufacturer selection is session-local in the wizard React state. ADR 014 event allowlist is unchanged; selected manufacturer may appear only in event `payload` (e.g. `{ "manufacturer": "samsung" }`).

Post-purchase redirect from buy success → setup. Detail page shows **Continue setup** when `setup_completed_at` is null and status is before `activated`.

Wizard persists `setup_resume_step` (1–4). Reopen resumes that step. `setup_version` records which wizard the user saw.

## API

| Method | Path | Notes |
|--------|------|-------|
| `POST` | `/api/v1/me/esims/{id}/events/` | Client telemetry; JWT + ownership; idempotent |
| `GET` | `/api/v1/me/esims/{id}/events/` | Chronological trail for owned eSIM |
| `GET` | `/api/v1/me/esims/{id}/` | Includes `status`, `activation_policy` snapshot, setup fields |

### Permissions

- JWT required.
- Object-level ownership via the same pattern as existing My eSIM endpoints (`OwnedEsimMixin`).
- Non-owner → `404` (match list/detail).
- Body must not override path ownership with `user_id` / foreign `esim_id`.

### Event POST body

```json
{
  "event_type": "install.qr_rendered",
  "idempotency_key": "client-stable-key",
  "setup_session_id": "uuid",
  "schema_version": 1,
  "payload": {}
}
```

Duplicate `idempotency_key` for the same eSIM returns the existing event (`200`).

## Wave 2+ backlog (document only)

Do not implement in Wave 1:

| Area | Notes |
|------|--------|
| Usage auto-refresh | Focus/interval polling on detail; optional Celery sync |
| Usage analytics | Snapshot deltas → used today / ETA days (`usage.*`) |
| Notification engine | Email: activated, low data, expiry, top-up (`notification.*`) |
| Connectivity test | Heuristic “Test my connection” |
| Support diagnostics | “Generate support report” (`support.*`) |
| Smart recommendations | Destination-based top-up/package suggestions |
| Travel timeline UI | Rich UI over `EsimLifecycleEvent` |
| Native home widget | Requires Flutter / mobile app |

## Definition of Done (Wave 1)

See measurable criteria in [ADR 014](../adr/014-esim-lifecycle-install-telemetry.md) and the Wave 1 implementation plan:

- Transition + unknown non-downgrade tests.
- Architecture test: only `LifecycleService` writes `Esim.status`.
- Idempotent events + ownership tests.
- Migration `unused` → `purchased`.
- Wizard iOS / Android / Desktop paths (Android: manufacturer picker + registry guides).
- OpenAPI coverage for events + new fields.

## Related

- [ADR 014](../adr/014-esim-lifecycle-install-telemetry.md)
- [RFC 001](./001-self-service-esim-flow.md)
- [ADR 005](../adr/005-domain-events.md)
