# OpenAPI changelog

Short history of the published RoamKit API schema (`roamkit-api/openapi/openapi.yaml`).
Documentation-only schema edits may appear here without an API version bump.

## 1.0.0

- Initial published OpenAPI 3 schema (C10).
- `drf-spectacular` infrastructure: `/api/schema/`, `/api/docs/`, `/api/redoc/`.
- Explicit `operationId`s, tags, JWT `bearerAuth`, CI validate + Spectral + path/security architecture tests.

## Unreleased (Faza 5 Wave 1)

- Additive: `GET|POST /api/v1/me/esims/{id}/events/` (install telemetry; JWT + ownership).
- Additive eSIM fields: `activation_policy`, `setup_*`.
- Coordinated sole-consumer enum rename: eSIM `status` `unused` → `purchased`; `active` → `activated`; new lifecycle values.
- Additive catalog field: `Package.activation_policy`.
