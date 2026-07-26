# OpenAPI changelog

Short history of the published RoamKit API schema (`roamkit-api/openapi/openapi.yaml`).
Documentation-only schema edits may appear here without an API version bump.

## 1.0.0

- Initial published OpenAPI 3 schema (C10).
- `drf-spectacular` infrastructure: `/api/schema/`, `/api/docs/`, `/api/redoc/`.
- Explicit `operationId`s, tags, JWT `bearerAuth`, CI validate + Spectral + path/security architecture tests.
